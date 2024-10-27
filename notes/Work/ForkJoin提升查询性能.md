# Java并行处理优化实践：从串行到ForkJoin的性能提升之路

## 一、问题背景

最近在处理期权询价数据时，遇到了一个性能瓶颈。系统需要处理大量的期权询价数据，这些数据具有层级结构，为了简化，代码采用三层结构：标的 -> 期限 -> 结构。每一层都需要进行数据分组和转换，并按照产品、运营要求进行排序，当数据量达到一定规模时（10个标的，每个标的1000条数据），串行处理的性能明显不足。

## 二、实践过程
### 2.1  数据结构抽象

在开始具体的处理逻辑之前，首先需要设计合适的数据结构。为方便展示，简化采用自顶向下三层嵌套结构，每层存储相应的数据和对下层的引用。这种结构清晰地反映了业务模型的层级关系，便于后续的并行处理。

为了模拟真实环境中的数据处理开销，在每个构造函数中添加了1毫秒的延时。在实际业务中，这里可能是数据转换、计算或者其他业务处理逻辑。

```java
public class DataModels {
    @Data // 使用Lombok简化getter/setter
    public static class StructureData {
        private String structureName;
        private List<OptionInquiry> inquiries;

        public StructureData(String structureName, List<OptionInquiry> inquiries) {
            this.structureName = structureName;
            this.inquiries = inquiries;
          	// 构造函数中的延时模拟数据处理耗时，下面构造器相同
        }
    }

    @Data
    public static class TermData {
        private int term;
        private List<StructureData> structures;

        public TermData(int term, List<StructureData> structures) {
            this.term = term;
            this.structures = structures;
            ThreadUtil.sleep(1);
        }
    }

    @Data
    public static class AssetData {
        private String assetCode;
        private List<TermData> terms;

        public AssetData(String assetCode, List<TermData> terms) {
            this.assetCode = assetCode;
            this.terms = terms;
            ThreadUtil.sleep(1);
        }
    }
}
```

### 2.2 处理流程抽象
```java
public abstract class InquiryProcessor {
    // 公共的处理入口，定义了基本的处理流程
    protected List<DataModels.AssetData> processInquiries(List<OptionInquiry> inquiries) {
        // 按标的分组
        Map<String, List<OptionInquiry>> assetGroups = groupByAsset(inquiries);
        return processAssetGroups(assetGroups);
    }
		// 具体的处理策略由子类实现
    protected abstract List<DataModels.AssetData> processAssetGroups(Map<String, List<OptionInquiry>> assetGroups);
		
  	// 提供通用的分组工具方法
    protected Map<String, List<OptionInquiry>> groupByAsset(List<OptionInquiry> inquiries) {
        return inquiries.stream().collect(Collectors.groupingBy(OptionInquiry::getUnderlyingAsset));
    }

    protected Map<Integer, List<OptionInquiry>> groupByTerm(List<OptionInquiry> inquiries) {
        return inquiries.stream().collect(Collectors.groupingBy(OptionInquiry::getTerm));
    }

    protected Map<String, List<OptionInquiry>> groupByStructure(List<OptionInquiry> inquiries) {
        return inquiries.stream().collect(Collectors.groupingBy(OptionInquiry::getStructure));
    }
}
```

#### 2.2.1 串行处理

最初的串行实现采用了简单直观的嵌套循环方式，代码易于理解但性能较差。这个实现可以作为性能基准，用来评估后续优化的效果。

```java
public static List<AssetData> convertToThreeLayers(List<OptionInquiry> inquiries) {
    List<AssetData> result = new ArrayList<>();
    
    // 按标的分组
    Map<String, List<OptionInquiry>> assetGroups = inquiries.stream()
            .collect(Collectors.groupingBy(i -> i.underlyingAsset));

    for (Map.Entry<String, List<OptionInquiry>> assetEntry : assetGroups.entrySet()) {
        // 按期限分组
        Map<Integer, List<OptionInquiry>> termGroups = assetEntry.getValue().stream()
                .collect(Collectors.groupingBy(i -> i.term));

        List<TermData> terms = new ArrayList<>();
        for (Map.Entry<Integer, List<OptionInquiry>> termEntry : termGroups.entrySet()) {
            // 按结构分组
            Map<String, List<OptionInquiry>> structureGroups = termEntry.getValue().stream()
                    .collect(Collectors.groupingBy(i -> i.structure));
                    
            List<StructureData> structures = structureGroups.entrySet().stream()
                    .map(e -> new StructureData(e.getKey(), e.getValue()))
                    .collect(Collectors.toList());

            terms.add(new TermData(termEntry.getKey(), structures));
        }

        result.add(new AssetData(assetEntry.getKey(), terms));
    }
    return result;
}
```

#### 2.2.2 ForkJoin优化并行处理

使用ForkJoin框架进行并行处理是一个自然的选择，因为我们的数据天然具有层级结构。这里采用了两层任务拆分策略：

- 第一层：将不同标的的处理拆分为独立任务
- 第二层：将每个标的下的期限处理拆分为子任务

这种策略既能充分利用多核性能，又不会产生过多的任务调度开销。

```java
public class ParallelInquiryProcessor extends InquiryProcessor {
    private final ForkJoinPool pool;

    public ParallelInquiryProcessor() {
        this.pool = ForkJoinPool.commonPool();
    }

    @Override
    protected List<DataModels.AssetData> processAssetGroups(Map<String, List<OptionInquiry>> assetGroups) {
        return pool.submit(() -> {
            List<ForkJoinTask<DataModels.AssetData>> assetTasks = assetGroups.entrySet().stream()
                    .map(entry -> new AssetProcessTask(entry.getKey(), entry.getValue()).fork())
                    .collect(Collectors.toList());

            return assetTasks.stream()
                    .map(ForkJoinTask::join)
                    .collect(Collectors.toList());
        }).join();
    }

    private static class AssetProcessTask extends RecursiveTask<DataModels.AssetData> {
        private final String assetCode;
        private final List<OptionInquiry> assetInquiries;

        public AssetProcessTask(String assetCode, List<OptionInquiry> assetInquiries) {
            this.assetCode = assetCode;
            this.assetInquiries = assetInquiries;
        }

        @Override
        protected DataModels.AssetData compute() {
            Map<Integer, List<OptionInquiry>> termGroups = assetInquiries.stream()
                    .collect(Collectors.groupingBy(OptionInquiry::getTerm));

            List<ForkJoinTask<DataModels.TermData>> termTasks = termGroups.entrySet().stream()
                    .map(entry -> new TermProcessTask(entry.getKey(), entry.getValue()).fork())
                    .collect(Collectors.toList());

            List<DataModels.TermData> terms = termTasks.stream()
                    .map(ForkJoinTask::join)
                    .collect(Collectors.toList());

            return new DataModels.AssetData(assetCode, terms);
        }
    }

    private static class TermProcessTask extends RecursiveTask<DataModels.TermData> {
        private final int term;
        private final List<OptionInquiry> termInquiries;

        public TermProcessTask(int term, List<OptionInquiry> termInquiries) {
            this.term = term;
            this.termInquiries = termInquiries;
        }

        @Override
        protected DataModels.TermData compute() {
            Map<String, List<OptionInquiry>> structureGroups = termInquiries.stream()
                    .collect(Collectors.groupingBy(OptionInquiry::getStructure));

            List<DataModels.StructureData> structures = structureGroups.entrySet().stream()
                    .map(e -> new DataModels.StructureData(e.getKey(), e.getValue()))
                    .collect(Collectors.toList());

            return new DataModels.TermData(term, structures);
        }
    }
}
```

## 三、性能测试

为了准确评估优化效果，我们需要构建完整的测试环境。测试代码不仅要能够准确测量性能差异，还要能够验证不同处理方式的结果一致性。

### 3.1 构建测试类

```java
public class PerformanceTest {
    public static void main(String[] args) {
        Instant start = Instant.now();
        System.out.println("开始处理期权询价数据...");

        // 获取标的列表
        List<String> assets = DataGenerator.getUnderlyingAssets();
        System.out.println("获取标的列表: " + assets.size() + "个标的");
        Instant afterGetAssets = Instant.now();

        // 查询询价数据
        List<OptionInquiry> inquiries = DataGenerator.generateTestData();
        System.out.println("查询到询价数据: " + inquiries.size() + "条");
        Instant afterQueryData = Instant.now();

        // 串行处理
        SerialInquiryProcessor serialProcessor = new SerialInquiryProcessor();
        Instant startSerial = Instant.now();
        List<DataModels.AssetData> serialResult = serialProcessor.processInquiries(inquiries);
        Instant afterSerial = Instant.now();

        // 并行处理
        ParallelInquiryProcessor parallelProcessor = new ParallelInquiryProcessor();
        Instant startParallel = Instant.now();
        List<DataModels.AssetData> parallelResult = parallelProcessor.processInquiries(inquiries);
        Instant afterParallel = Instant.now();

        // 打印统计信息
        printStatistics(start, afterGetAssets, afterQueryData,
                startSerial, afterSerial, startParallel, afterParallel);
      	validateResults(serialResult, parallelResult);
    }

    private static void printStatistics(Instant... timestamps) {
        System.out.println("\n处理完成，统计信息：");
        System.out.println("获取标的列表耗时: " + Duration.between(timestamps[0], timestamps[1]).toMillis() + "ms");
        System.out.println("查询询价数据耗时: " + Duration.between(timestamps[1], timestamps[2]).toMillis() + "ms");
        System.out.println("串行处理耗时: " + Duration.between(timestamps[3], timestamps[4]).toMillis() + "ms");
        System.out.println("并行处理耗时: " + Duration.between(timestamps[5], timestamps[6]).toMillis() + "ms");
        System.out.println("总耗时: " + Duration.between(timestamps[0], timestamps[6]).toMillis() + "ms");
    }
  
}

```

验证结果一致性

```java
private static void validateResults(List<DataModels.AssetData> serialResult,
                                   List<DataModels.AssetData> parallelResult) {
   // 先比较整体大小
   if (serialResult.size() != parallelResult.size()) {
       System.err.println("结果数量不一致：串行=" + serialResult.size()
               + ", 并行=" + parallelResult.size());
       return;
   }

   // 逐个资产比较
   for (int i = 0; i < serialResult.size(); i++) {
       DataModels.AssetData serial = serialResult.get(i);
       DataModels.AssetData parallel = parallelResult.get(i);

       // 比较资产码
       if (!serial.getAssetCode().equals(parallel.getAssetCode())) {
           System.err.println("资产" + i + "不一致");
           return;
       }

       // 比较期限
       List<DataModels.TermData> serialTerms = serial.getTerms();
       List<DataModels.TermData> parallelTerms = parallel.getTerms();
       if (serialTerms.size() != parallelTerms.size()) {
           System.err.println("资产" + serial.getAssetCode() + "的期限数量不一致");
           return;
       }

       // 比较每个期限下的结构
       for (int j = 0; j < serialTerms.size(); j++) {
           DataModels.TermData serialTerm = serialTerms.get(j);
           DataModels.TermData parallelTerm = parallelTerms.get(j);

           if (serialTerm.getTerm() != parallelTerm.getTerm()) {
               System.err.println("资产" + serial.getAssetCode() + "期限值不一致");
               return;
           }

           List<DataModels.StructureData> serialStructures = serialTerm.getStructures();
           List<DataModels.StructureData> parallelStructures = parallelTerm.getStructures();
           
           if (serialStructures.size() != parallelStructures.size()) {
               System.err.println("资产" + serial.getAssetCode() + "期限" + serialTerm.getTerm() + "的结构数量不一致");
               return;
           }

           // 比较每个结构
           for (int k = 0; k < serialStructures.size(); k++) {
               DataModels.StructureData serialStructure = serialStructures.get(k);
               DataModels.StructureData parallelStructure = parallelStructures.get(k);

               if (!serialStructure.getStructureName().equals(parallelStructure.getStructureName())) {
                   System.err.println("资产" + serial.getAssetCode() + "期限" + serialTerm.getTerm() 
                           + "结构" + serialStructure.getStructureName() + "名称不一致");
                   return;
               }

               // 比较询价数据
               List<OptionInquiry> serialInquiries = serialStructure.getInquiries();
               List<OptionInquiry> parallelInquiries = parallelStructure.getInquiries();
               
               if (serialInquiries.size() != parallelInquiries.size()) {
                   System.err.println("资产" + serial.getAssetCode() + "期限" + serialTerm.getTerm() 
                           + "结构" + serialStructure.getStructureName() + "询价数量不一致");
                   return;
               }

               // 比较每条询价数据
               for (int m = 0; m < serialInquiries.size(); m++) {
                   OptionInquiry serialInquiry = serialInquiries.get(m);
                   OptionInquiry parallelInquiry = parallelInquiries.get(m);
                   
                   if (serialInquiry.getPrice() != parallelInquiry.getPrice()) {
                       System.err.println("资产" + serial.getAssetCode() + "期限" + serialTerm.getTerm() 
                               + "结构" + serialStructure.getStructureName() + "第" + m + "条询价数据价格不一致");
                       return;
                   }
               }
           }
       }
   }

   System.out.println("验证通过：串行和并行处理结果完全一致");
}
```



### 3.2 构建测试依赖的数据

为了使测试更接近真实场景，我们生成了具有一定规模和复杂度的测试数据。每个标的都有多个期限，每个期限下有多个结构，以模拟真实的业务场景。

```java
public class DataGenerator {
    private static final List<String> UNDERLYING_ASSETS = Arrays.asList(
            "600000.SH", "600036.SH", "601318.SH", "601398.SH", "601988.SH",
            "000001.SZ", "000651.SZ", "000858.SZ", "600519.SH", "601166.SH"
    );

    public static List<String> getUnderlyingAssets() {
        return UNDERLYING_ASSETS;
    }

    public static List<OptionInquiry> generateTestData() {
        List<OptionInquiry> inquiries = new ArrayList<>();
        Random random = new Random();

        for (String asset : UNDERLYING_ASSETS) {
            int eachLevelSize = 10;
            for (int term = 1; term <= eachLevelSize; term++) {
                for (int structureId = 1; structureId <= eachLevelSize; structureId++) {
                    // 假设每个维度（标的、期限、结构）有十个报价
                    for (int i = 0; i < 10; i++) {
                        String structure = "Structure" + structureId;
                        double price = 100 + random.nextDouble() * 100;
                        inquiries.add(new OptionInquiry(asset, term, structure, price));
                    }
                }
            }
        }
        return inquiries;
    }
}
```



### 3.3 测试结果

在处理10,000条数据时，性能提升8倍左右，控制台打印结果：

```
获取标的列表: 10个标的
查询到询价数据: 10000条

处理完成，统计信息：
获取标的列表耗时: 9ms
查询询价数据耗时: 37ms
串行处理耗时: 1661ms
并行处理耗时: 237ms
总耗时: 1970ms
验证通过：串行和并行处理结果一致
```





## 四、结论

通过使用ForkJoin框架，成功将处理时间从1.6秒优化到了0.2秒，性能提升显著。
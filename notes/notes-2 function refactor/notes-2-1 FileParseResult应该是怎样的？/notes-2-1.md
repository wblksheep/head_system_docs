FileParseResult 这个类应该是怎样的，json 格式数据吗？该怎么创建呢？

```java
// 基类抽象
public abstract class HeadInventoryProcessor<T> {
    protected final FileParser fileParser;
    protected final StatusOperator statusOperator;

    public HeadInventoryProcessor(FileParser fileParser, StatusOperator statusOperator) {
        this.fileParser = fileParser;
        this.statusOperator = statusOperator;
    }

    // 模板方法
    public final PageBean<T> process(Integer pageNum, Integer pageSize, MultipartFile... files) {
        // 1. 解析文件
        FileParseResult parseResult = fileParser.parseFiles(files);

        // 2. 校验数据
        ValidationResult validationResult = validate(parseResult);

        // 3. 执行操作
        OperationResult operationResult = executeOperation(validationResult.getValidRecords());

        // 4. 构建响应
        return buildResponse(pageNum, pageSize,
            validationResult.getInvalidRecords(),
            operationResult.getFailedRecords());
    }

    protected abstract ValidationResult validate(FileParseResult parseResult);
    protected abstract OperationResult executeOperation(List<?> validRecords);
    protected abstract PageBean<T> buildResponse(Integer pageNum, Integer pageSize,
                                               List<?> invalidRecords, List<?> failedRecords);
}

// 文件解析策略接口
public interface FileParser {
    FileParseResult parseFiles(MultipartFile... files);
}
```

--- R1
`FileParseResult` 的设计应该是一个通用的中间数据结构，用于承载不同文件类型的解析结果。以下是推荐实现方案：

### 1. 基础类结构设计（不使用 JSON）

```java
public class FileParseResult {
    // 原始记录集合（Excel/TXT解析后的原始数据）
    private List<RawRecord> rawRecords = new ArrayList<>();

    // 转换后的DTO集合（校验前的中间态）
    private List<BaseDTO> convertedRecords = new ArrayList<>();

    // 解析错误信息（格式错误的行）
    private List<ParseError> parseErrors = new ArrayList<>();

    // 文件类型标识
    private FileType fileType;

    // 元数据（如文件名、解析时间等）
    private Map<String, Object> metadata = new HashMap<>();

    // 枚举定义
    public enum FileType {
        ALLOCATION_EXCEL,
        MAINTENANCE_EXCEL,
        INVENTORY_TXT
    }

    // 嵌套类定义
    public static class RawRecord {
        private int rowNumber;
        private Map<String, String> fields;

        public RawRecord(int rowNumber, Map<String, String> fields) {
            this.rowNumber = rowNumber;
            this.fields = fields;
        }
    }

    public static class ParseError {
        private int rowNumber;
        private String errorMessage;
        private String rawContent;
    }
}
```

### 2. 创建方式建议（建造者模式）

```java
public class FileParseResultBuilder {
    private final FileParseResult result = new FileParseResult();

    public FileParseResultBuilder withFileType(FileType type) {
        result.fileType = type;
        return this;
    }

    public FileParseResultBuilder addRawRecord(int rowNum, Map<String, String> fields) {
        result.rawRecords.add(new RawRecord(rowNum, fields));
        return this;
    }

    public FileParseResultBuilder addConvertedRecord(BaseDTO dto) {
        result.convertedRecords.add(dto);
        return this;
    }

    public FileParseResultBuilder addParseError(int rowNum, String error, String raw) {
        result.parseErrors.add(new ParseError(rowNum, error, raw));
        return this;
    }

    public FileParseResult build() {
        // 添加自动生成的元数据
        result.metadata.put("parseTime", LocalDateTime.now());
        return result;
    }
}
```

### 3. 具体解析器实现示例（Excel 解析器）

```java
@Component
@RequiredArgsConstructor
public class AllocationExcelParser implements FileParser {
    private final DTOConverter converter;

    @Override
    public FileParseResult parseFiles(MultipartFile... files) {
        FileParseResultBuilder builder = new FileParseResultBuilder()
                .withFileType(FileType.ALLOCATION_EXCEL);

        try (Workbook workbook = new XSSFWorkbook(files.getInputStream())) {
            Sheet sheet = workbook.getSheetAt(0);

            for (Row row : sheet) {
                try {
                    Map<String, String> fields = extractRowData(row);
                    builder.addRawRecord(row.getRowNum(), fields);

                    // 转换为中间DTO
                    SprinklerAllocationDTO dto = converter.convert(fields);
                    builder.addConvertedRecord(dto);
                } catch (ParseException e) {
                    builder.addParseError(row.getRowNum(), e.getMessage(), row.toString());
                }
            }
        } catch (IOException e) {
            throw new FileParseException("Excel文件读取失败", e);
        }

        return builder.build();
    }

    private Map<String, String> extractRowData(Row row) {
        // 具体解析逻辑...
    }
}
```

### 4. 使用场景示例

```java
// 在 Processor 中使用
protected ValidationResult validate(FileParseResult parseResult) {
    List<BaseDTO> validRecords = new ArrayList<>();
    List<ValidationError> errors = new ArrayList<>();

    for (BaseDTO dto : parseResult.getConvertedRecords()) {
        if (validateDTO(dto)) {
            validRecords.add(dto);
        } else {
            errors.add(new ValidationError(dto, "校验失败原因"));
        }
    }

    return new ValidationResult(validRecords, errors);
}
```

### 5. 配套结构设计

```java
// DTO 基类
public abstract class BaseDTO {
    protected String sprinklerNo;
    protected SprinklerType type;
    // 公共字段...
}

// 校验结果
public class ValidationResult {
    private List<BaseDTO> validRecords;
    private List<ValidationError> invalidRecords;
}

// 操作结果
public class OperationResult {
    private List<BaseDTO> successRecords;
    private List<OperationError> failedRecords;
}
```

### 关键设计点说明：

1. **多层级数据承载**：

   - `rawRecords`：保留原始解析数据用于审计
   - `convertedRecords`：转换后的业务对象
   - `parseErrors`：格式层面的错误信息

2. **类型安全**：

   - 通过泛型设计确保不同解析器返回类型安全的结果

   ```java
   public interface FileParser<T extends BaseDTO> {
       FileParseResult<T> parseFiles(MultipartFile... files);
   }
   ```

3. **扩展性**：

   - 通过`metadata`字段承载扩展信息（如文件哈希值、解析耗时等）
   - 支持后续添加新的文件类型

4. **错误追踪**：

   - 精确记录错误发生的行号和原始内容
   - 区分格式错误（ParseError）和业务校验错误（ValidationError）

5. **审计能力**：
   ```java
   // 记录完整解析轨迹
   result.metadata.put("fileHash", DigestUtils.md5Hex(file.getBytes()));
   result.metadata.put("operator", SecurityUtils.getCurrentUser());
   ```

建议配套实现一个可视化诊断工具：

```java
@RestController
@RequiredArgsConstructor
public class ParseDebugController {
    private final FileParseResultRepository resultRepository;

    @GetMapping("/parse-results/{id}")
    public FileParseResultDebugVO getResult(@PathVariable String id) {
        FileParseResult result = resultRepository.findById(id);

        return new FileParseResultDebugVO(
                result.getFileType(),
                result.getMetadata(),
                result.getRawRecords(),
                result.getConvertedRecords(),
                result.getParseErrors()
        );
    }
}
```

这种设计可以满足：

1. 问题追溯：通过 rowNumber 定位原始数据位置
2. 数据完整性：保留原始数据和转换后数据的映射关系
3. 性能监控：通过 metadata 统计解析耗时
4. 格式兼容：轻松支持新增文件类型

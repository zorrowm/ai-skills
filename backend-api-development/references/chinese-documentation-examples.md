# 中文注释与 Hprose 示例

以下示例面向模板中的 `Grade` 模型和 `BackAPIService.Utils.PageList<T>`。示例为新接口选择了页码从 1 开始、每页 1～200 条的约束，并在 Service 中实现校验。这些值是示例设计，不代表模板已有行为；用于存量接口时应保留并说明真实契约。

## HTTP API：用途、全部参数、返回值均有中文说明

```csharp
using BackAPIService.Services;
using BackAPIService.Utils;
using Microsoft.AspNetCore.Mvc;
using Models;

namespace BackAPIService.Controllers;

/// <summary>
/// 提供年级信息查询接口。
/// </summary>
[ApiController]
[Route("api/[controller]/[Action]")]
public class GradeController : ControllerBase
{
    private readonly GradeService _gradeService;

    public GradeController(GradeService gradeService)
    {
        _gradeService = gradeService;
    }

    /// <summary>
    /// 按年级名称关键字查询年级分页列表。
    /// </summary>
    /// <param name="keyword">年级名称关键字；可省略，null 或空字符串表示不按名称筛选。</param>
    /// <param name="pageIndex">页码；从 1 开始，默认值为 1。</param>
    /// <param name="pageSize">每页记录数；取值为 1～200，默认值为 20。</param>
    /// <returns>
    /// 年级分页结果：Datas 为当前页数据，TotalCount 为符合条件的总记录数；
    /// 未匹配记录时 Datas 为空列表且 TotalCount 为 0，页码超出范围时 Datas 为空列表。
    /// </returns>
    /// <remarks>结果按年级 ID 升序排列；输入范围校验由服务层执行。</remarks>
    /// <exception cref="ArgumentOutOfRangeException">页码小于 1，或每页记录数超出 1～200。</exception>
    [HttpGet]
    public Task<PageList<Grade>> GetGradeListAsync(
        [FromQuery] string keyword = null,
        [FromQuery] int pageIndex = 1,
        [FromQuery] int pageSize = 20)
    {
        return _gradeService.GetGradeListAsync(keyword, pageIndex, pageSize);
    }
}
```

这里记录异常触发条件，未承诺 HTTP 400。若项目要求返回 400，应先通过已生效的异常映射或明确的 HTTP 处理实现该行为，再补响应文档。

## Service：完整 XML 文档与 Hprose 说明同时存在

```csharp
using BackAPIService.Utils;
using Microsoft.Extensions.DependencyInjection;
using Models;
using SqlSugar;
using XFrameBack.Web.GCBridge.Hprose;

namespace BackAPIService.Services;

/// <summary>
/// 提供年级查询业务，供 HTTP API 与 Hprose 调用。
/// </summary>
[Service("年级查询服务")]
public class GradeService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public GradeService(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    /// <summary>
    /// 按名称关键字筛选年级，并按 ID 升序返回指定页。
    /// </summary>
    /// <param name="keyword">年级名称关键字；null 或空字符串表示不按名称筛选。</param>
    /// <param name="pageIndex">页码；必须大于或等于 1。</param>
    /// <param name="pageSize">每页记录数；必须为 1～200。</param>
    /// <returns>
    /// 包含当前页年级数据 Datas 与总记录数 TotalCount 的分页结果；
    /// 无匹配记录时 Datas 为空列表且 TotalCount 为 0，超出总页数时 Datas 为空列表。
    /// </returns>
    /// <remarks>
    /// 输入约束在服务层统一校验，使 HTTP 与 Hprose 使用同一套业务规则。
    /// 使用模板既有分页工厂；其 TotalPages 字段尚未计算，本接口不以该字段表达总页数。
    /// </remarks>
    /// <exception cref="ArgumentOutOfRangeException">页码小于 1，或每页记录数超出 1～200。</exception>
    [Hprose("按名称关键字查询年级分页列表",
        "keyword：年级名称关键字；null 或空字符串表示不按名称筛选",
        "pageIndex：页码；必须大于或等于 1",
        "pageSize：每页记录数；必须为 1～200")]
    public async Task<PageList<Grade>> GetGradeListAsync(
        string keyword, int pageIndex, int pageSize)
    {
        if (pageIndex < 1)
        {
            throw new ArgumentOutOfRangeException(nameof(pageIndex), "页码必须大于或等于 1。");
        }

        if (pageSize < 1 || pageSize > 200)
        {
            throw new ArgumentOutOfRangeException(nameof(pageSize), "每页记录数必须为 1～200。");
        }

        using var scope = _scopeFactory.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<ISqlSugarClient>();
        var query = db.Queryable<Grade>();

        if (!string.IsNullOrEmpty(keyword))
        {
            query = query.Where(grade => grade.Name.Contains(keyword));
        }

        // 按唯一 ID 排序，为记录集合不变时的分页提供确定顺序。
        // 取得本页结果后再释放 scope，避免查询使用已释放的数据库服务。
        return await PageList<Grade>.CreateAsync(
            query.OrderBy(grade => grade.Id), pageIndex, pageSize);
    }
}
```

此处为了适配现有分页类型使用了原分页工厂；实际优化异步查询时，先核对 SqlSugar 版本支持的 API，再实现原生异步分页，保持返回契约一致。

若需要把 `TotalPages` 纳入有效返回字段，应先实现并验证总页数计算，再更新接口说明。不要在注释中假装已有计算。

## Hprose 参数说明的三条规则

下面只展示标注语法；方法是否成功、返回值语义和输入约束，应按实际实现填写。精简 Hprose 说明时，XML `<param>` 仍须写全所有参数，`<returns>` 仍须完整描述结果。

### 1. 多个参数说明合并在一个字符串内，最后一个字符串说明返回值

假设方法签名为 `Task<bool> SaveImageDataFile(string pID, string path, string code = null)`，其中 `pID`、`path` 必填，`code` 可选：

```csharp
[Hprose("文件夹添加数据",
    "pID：项目名称；必填,path：文件夹路径；必填,code：业务编码；可选",
    "返回值：true 表示保存成功，false 表示未保存")]
```

参数说明之间使用逗号 `,`；每个参数说明内部需要停顿时使用分号。最后一个字符串以“返回值：”开头，清楚区分返回值和输入参数。

### 2. 一次传入多个字符串，每个字符串对应一个参数说明

假设方法签名为 `Task<bool> UpdateImageInfoShootingDateAsync(string dsName, string shootingdate, int type = 0, string filter = null)`：

```csharp
[Hprose("更改所选影像的时相",
    "dsName：数据集名称；必填",
    "shootingdate：时相；必填，格式按实际业务约定",
    "type：类型；可选，默认 0",
    "filter：过滤条件；可选，null 表示不附加过滤条件")]
```

方法用途后的字符串按参数顺序排列。此写法的每项用于参数说明，返回值另在 XML `<returns>` 中完整描述。

### 3. 参数说明可以只覆盖必填参数

上面的 `SaveImageDataFile` 也可省略可选参数 `code` 的 Hprose 说明。合并写法仍保留最后的返回值说明：

```csharp
[Hprose("文件夹添加数据",
    "pID：项目名称；必填,path：文件夹路径；必填",
    "返回值：true 表示保存成功，false 表示未保存")]
```

逐项写法同样可以仅说明必填的 `pID`、`path`，此时两个参数说明使用两个字符串：

```csharp
[Hprose("文件夹添加数据", "pID：项目名称；必填", "path：文件夹路径；必填")]
```

精简后不检查“Hprose 说明条数等于全部形参数量”，而检查必填参数是否覆盖、名称是否对应，以及已说明参数的顺序是否与签名一致。默认值和业务校验共同决定必填性，不能仅凭 `= null` 判定业务上一定可选。

## 关键算法：说明计算依据、边界与原因

以下是可用于分页工具的独立辅助算法。它没有修改模板既有 `PageList<T>`，也不应添加 Hprose 标记。

```csharp
/// <summary>
/// 按每页记录数计算容纳全部记录所需的总页数。
/// </summary>
/// <param name="totalCount">总记录数；必须大于或等于 0。</param>
/// <param name="pageSize">每页记录数；必须大于 0，本方法不接受 0 表示不分页。</param>
/// <returns>总页数；无记录时返回 0，有不足一页的剩余记录时增加一页。</returns>
/// <remarks>
/// 采用整数商与余数实现向上取整，避免浮点精度误差；
/// 不采用先加 pageSize - 1 再除的写法，避免大记录数加法溢出。
/// 时间复杂度和额外空间复杂度均为 O(1)。
/// </remarks>
/// <exception cref="ArgumentOutOfRangeException">总记录数为负数，或每页记录数不大于 0。</exception>
private static long CalculateTotalPages(long totalCount, int pageSize)
{
    if (totalCount < 0)
    {
        throw new ArgumentOutOfRangeException(nameof(totalCount), "总记录数不能为负数。");
    }

    if (pageSize <= 0)
    {
        throw new ArgumentOutOfRangeException(nameof(pageSize), "每页记录数必须大于 0。");
    }

    var completePages = totalCount / pageSize;

    // 余数不为零表示最后一页未填满，仍须计为一页。
    return completePages + (totalCount % pageSize == 0 ? 0 : 1);
}
```

涉及该算法的行为变更时，至少验证：零记录、整页、余数一条、非法输入，以及 `long.MaxValue` 记录数。

## 返回值与字段说明的写法

| 实际实现 | 应说明的业务语义 |
| --- | --- |
| `ExecuteCommandAsync() > 0` 的删除结果 | `true` 表示至少删除一条记录，`false` 表示没有记录被删除；除非另有校验，不能说明全部指定 ID 均已删除 |
| `FirstAsync()` 查询单对象 | 按所用 ORM 的实际行为说明未找到是 `null` 还是抛异常，不把单对象写成列表 |
| 文件下载并在未找到时返回 404 | 成功时返回档案文件及实际 MIME 类型，未找到时返回 HTTP 404；上传参数另说明 `multipart/form-data` 与字段含义 |
| 时间、金额、坐标等字段 | 中文名称并补实际格式、时区、单位或坐标系，信息不足时先查实现与调用方 |
| 多表写入 | 描述实际影响的对象、成功判定和事务边界，不能用“保存成功”掩盖部分写入风险 |

## Swagger XML 配置示意

目标项目已经生成 XML，且实际启动仅调用默认 `AddSwaggerGen()` 时，可在**原注册位置**按需补充：

```csharp
builder.Services.AddSwaggerGen(options =>
{
    var assemblyName = typeof(Program).Assembly.GetName().Name;
    var xmlPath = Path.Combine(AppContext.BaseDirectory, $"{assemblyName}.xml");
    options.IncludeXmlComments(xmlPath, includeControllerXmlComments: true);
});
```

保留已有的安全定义、文档分组等配置。构建后确认 XML 文件存在，再检查实际 Swagger 文档；不要在缺文件时静默跳过而宣称文档接入完成。其他程序集中的 DTO 需要按项目布局加载对应 XML。

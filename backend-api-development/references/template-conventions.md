# 模板事实与适配边界

本 Skill 提取自 `BackAPIServiceTemplate/NET10`。以下是提取时核对的事实；迁移到其他项目或升级依赖后，应以目标源码和实际运行配置为准。

## 事实来源

| 模板文件 | 已核对的约定 | 开发时的应用 |
| --- | --- | --- |
| `BackAPIService.csproj` | `net10.0`、隐式 using、关闭可空检查；启用 XML 文档生成；Debug / Release 均屏蔽 `1591` | 中文说明需单独验收，不能用“没有文档警告”作为完整证明；迁移时跟随目标可空配置 |
| `Controllers/StudentController.cs` 等 | `[ApiController]`、`api/[controller]/[Action]` 路由、构造注入 Service；接口主要直接返回业务类型 | 保持既有路由和响应契约；核对实际 Action 名称，框架可能移除 `Async` 后缀 |
| `Services/StudentService.cs` | `[Service]`、`[Hprose]`、方法内创建 scope 并取得 `ISqlSugarClient` | 保留服务生命周期；XML 覆盖全部参数，Hprose 可按需只说明必填参数 |
| `Program.cs` | `AddHproseOptions`、`AddHproseServices(assembly, "Services")`、`UseHproseMiddleware`；直接调用 `AddSwaggerGen()` | 确认真实注册链，不能只根据扩展文件存在推断它已启用 |
| `GlobalUsings.cs` | 全局引入 `BackAPIService.Utils`、`Models`、`SqlSugar` 和 Hprose 命名空间 | 当前 Service 的 `PageList<T>` 解析为 Utils 版本 |
| `Configs/hprose.json` | HTTP 路由 `/hprose`，WebSocket 路由 `/wshprose` | 路由是可配置事实，不硬编码为所有项目必须使用的地址 |
| `Utils/PageList.cs` | 使用 `TotalCount`、`TotalPages`、`Datas`；`pageIndex == 0` 或 `pageSize == 0` 时不分页，否则按从 1 开始的页码计算偏移 | 存量接口的零值语义需写清楚；不要擅自改成从 0 开始的分页 |
| `Base/PageList.cs` | 另一个同名类型，字段为 `resultList`、`pageIndex`、`pageSize` 等 | 与 Utils 版本不同，不能混写返回字段 |
| `Base/JsonView.cs` | 存在 `Code`、`Msg`、`Count`、`Data` 的包装类型 | 存在该类型不代表所有接口使用它 |
| `Extensions/SwaggerUIExtensions.cs` | `AddSwaggerService` 内含 `IncludeXmlComments`；模板启动代码并未调用此扩展 | 不宣称 Swagger 已加载 XML；需要文档展示时核对并调整实际注册 |
| `Filters/` | 包含多种异常过滤器 / 处理器文件；部分命名空间来自其他项目 | 先核对注册与路由处理链，不宣称已有统一错误响应 |

## 已核对的 Hprose 特性

模板依赖 `XFrameBack.Web 1.0.8`。通过本地依赖程序集反射，确认命名空间 `XFrameBack.Web.GCBridge.Hprose` 中具有以下构造签名：

```csharp
HproseAttribute(string method, string description, string chinesedesc)
HproseAttribute(string methodDesc, params string[] paramsDesc)
ServiceAttribute(string serviceDescription)
ServiceAttribute(string serviceDescription, string prefix)
```

`HproseAttribute` 公开 `MethodDescription` 和 `ParamsDescription` 属性；服务特性公开 `Description` 与 `Prefix`。

本项目支持以下编写规则：

1. **合并说明**：第二个字符串用逗号分隔多个参数说明，最后一个字符串为返回值说明。
2. **逐项说明**：方法用途后传入多个字符串，每个字符串说明一个参数。
3. **精简说明**：两种格式均可仅覆盖必填参数，保持已说明参数在签名中的相对顺序；可选参数在 XML 中仍须完整说明。

合并写法：

```csharp
[Hprose("更新档案内容",
    "id：档案 ID；必填,name：含扩展名的文件名；必填,datas：文件内容字节数组",
    "返回值：true 表示更新成功，false 表示没有记录被更新")]
```

逐项写法：

```csharp
[Hprose("更新档案内容",
    "id：档案 ID；不能为空",
    "name：含扩展名的文件名；不能为空",
    "datas：文件内容的字节数组；空值规则以实际校验为准")]
```

可直接传多个字符串，也可沿用显式字符串数组；不要求数组长度等于形参数量。无参方法可只写 `[Hprose("方法的中文用途")]`。XML 文档始终完整说明所有参数和返回值。

### 当前版本的元数据行为

本地反射已核对：三字符串重载把第二、第三个字符串原样保存为 `ParamsDescription` 中的两个条目；`params string[]` 重载也按传入条目保存。三个字符串虽然会优先匹配固定参数重载，仍支持上述编写格式，无需为此强制改成数组。

因此，“逗号分隔参数、最后一项为返回值”是本项目的说明编写约定。当前特性本身不自动拆分逗号，也没有独立的返回值说明属性；需要在展示工具中拆分或识别返回值时，应另行核对其解析实现。

## 模板中需要改进的地方

这些是制定规范时发现的缺口，不是应复制到新代码中的范式：

- 部分 Controller 的 `<returns>`、`<param>` 为空，部分 Action 没有注释。
- Service 中缺少 XML 文档，部分公开业务方法缺少 Hprose 说明；已有列表方法只有方法描述，应按实际必填输入补充 Hprose 说明，XML 则补齐全部参数。
- 部分实体保留空白摘要和 `Desc:`、`Default:`、`Nullable:` 生成器文本，无法解释业务字段。
- Utils 分页类型的 `TotalPages` 属性在现有工厂方法中没有赋值；不能仅因属性存在就承诺它返回正确总页数。
- Utils 分页方法以 `Task.Run` 包装同步数据库查询，不宜作为新数据库异步方法的默认写法。
- 学生查询按记录逐个查询档案，存在逐行查询模式；新增相关功能时优先考虑关联或批量查询。
- 多表写入方法没有显式事务，不应在注释中承诺整组写入原子成功。
- XML 已生成，但启动处直接调用默认 `AddSwaggerGen()`；需另行核对中文说明是否实际进入 Swagger。

提取 Skill 本身不修复这些代码。后续按用户指定范围开发或补说明时，再处理对应缺口及其兼容性。

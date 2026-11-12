# OIE Web 管理支持

Web 管理支持 **1.3.0** 捆绑 Web 客户端 **1.3.0**，支持 OIE **4.5.2、4.6.0 和 4.7.0**。

完整的 [OIE Web 管理器](https://github.com/gibson9583/oie-web-client)，
作为一个标准扩展发布，引擎本身**无需任何修改**。它同时安装浏览器客户端和引擎侧的 API，
用于消息树、JavaScript 验证和扩展提供的 Web UI。

## 一个扩展

每个发布版本包含一个构建产物：

| 扩展 | 构建产物 | 功能 |
|---|---|---|
| **Web 管理支持** (`websupport`) | `websupport-<version>.zip` | 安装支持的 API 并将 `oie-webadmin.war` 部署到此 OIE 服务器的嵌入式 Jetty 中，在 `/oie-webadmin/` 路径下提供服务。 |

通过 Swing 管理器安装它（或者在已有 Web 客户端时通过其扩展页面安装），然后重启 OIE。
服务插件会在 Jetty 扫描 Web 应用之前将 `oie-webadmin.war` 复制到 `<OIE_HOME>/webapps/`。
重启后打开 `https://<host>:8443/oie-webadmin/`。

同一个客户端仍然可以使用 Node.js 或 Docker 运行。安装 Web 管理支持也会使嵌入式副本可用；
它不会阻止那些部署模式。

卸载 **Web 管理支持** 会在下次引擎重启时移除其已部署的 WAR（完成卸载的那次重启）。
如果在卸载待处理时引擎被强制终止，请手动删除 `<OIE_HOME>/webapps/oie-webadmin.war`。

## 提供的功能

所有端点位于 `/api/extensions/websupport` 下，仅需认证会话：

| 端点 | 用途 |
|---|---|
| `POST /datatypes/_serialize?dataType=&props=` | 通过引擎**自身的数据类型序列化器**序列化消息——输出与运行时 `msg`/`tmp` 字节相同——加上数据类型的词汇描述用于消息树注释。 |
| `POST /javascript/_validate` | 使用引擎的 **Rhino** 编译脚本，返回 `{ error: string\|null }`——与运行时编译完全匹配的验证。 |
| `POST /elements/_responseVariables` | 针对传入的未保存设置调用已安装规则与步骤的 `getResponseVariables()`，返回 `{ responseVariables: string[] }`。 |
| `GET /webplugins` | 列出已安装、已启用且附带 Web UI 半部分的扩展（`webadmin/plugin.json`）。 |
| `GET /webplugins/{extension}/{path}` | 从扩展的 `webadmin/` 文件夹提供静态文件，使插件的浏览器 UI 跟随其安装的引擎。 |

Web 管理器在会话开始时探测这些端点——优先使用引擎原生端点（内置了这些端点的引擎），
然后回退到此插件——并在两者都不存在时优雅降级（无消息树、无服务器端验证、无引擎提供的插件 UI）。

响应变量发现接受引擎的 XML 列表或等价的以类名为键的 JSON
（`{"list":{"your.plugin.CustomStep":{"@version":"4.6.0", ...}}}`）。
请发送完整的引擎元素记录，并为每个元素包含 `sequenceNumber`；
引擎在反序列化时会按该字段对多元素列表排序。
调用方提供已启用的源元素和全部目标元素，与 Swing 的包含规则一致。
空列表以及不返回任何变量的提供程序都是有效的；无效元素或失败的提供程序会使请求失败，
而不是返回不完整的结果。不会保存任何通道，也不会执行生成的 JavaScript。
此端点需要更新后的 Web 管理支持插件；客户端必须将其缺失与成功的空结果区分处理。

## 来自数据类型扩展的词汇描述

Swing 从每个数据类型的 `getVocabulary()` 获取消息树描述。
要在服务器端暴露同样的词汇，已启用的扩展可以在其现有的 `webadmin/plugin.json` 中声明：

```json
{
  "id": "datatype-edifact",
  "vocabularies": {
    "EDIFACT": "com.mirth.connect.plugins.datatypes.edifact.EDIFACTVocabulary"
  }
}
```

将 `vocabularies` 合并进现有清单；保留其余字段。每个键必须与已安装数据类型的插件点名称匹配。
每个类必须是引擎 SHARED 库中的 `MessageVocabulary` 子类，具有公共的
`(String version, String type)` 构造函数和匹配的 `getDataType()`。
构造函数接收该消息的序列化器元数据，与 Swing 一致。
服务器上不会实例化任何 Swing 客户端插件。

现有的 HL7 v2、X12、NCPDP 和 DICOM 映射仍然是权威来源。扩展发现仅包含引擎已启用的扩展；
`"enabled": false` 的清单不贡献词汇。缺失、无效、不可用或冲突的声明会回退为纯标签，
且不会阻止消息序列化。清单在每次请求时读取，词汇实例从不共享，
因此启用发现或声明的变化不会留下过期结果。安装更新后的 SHARED 库仍需引擎正常重启。
没有此声明的扩展保持其现有行为。

这使用现有的序列化端点和响应结构；无需更改客户端 API 版本。

## 安装——请先执行此操作

此插件是唯一无法通过社区商店自行安装的插件，因为商店自身的 UI 就是由它提供的。
引导顺序：

1. 从 [Releases](../../releases) 下载 `websupport-<version>.zip`。
2. 在 Web 管理器中：**扩展 → 安装**（扩展安装仅使用引擎核心 REST API，因此无需此插件即可工作）。
   经典 Swing 管理器同样可以。
3. 重启引擎。

之后，插件 UI（包括社区商店）将亮起，此插件会像其他扩展一样通过商店更新自身。

## 构建

使用原版 OIE **4.5.2**（最旧的受支持引擎）的安装 jar 包进行编译。
编译基线与受支持的引擎版本相互独立。打包必须使用已构建的 Web 客户端 **1.3.0** WAR：

```bash
cd ../oie-web-client
npm ci
npm run build:war

cd ../oie-web-support-plugin
OIE_HOME=/path/to/oie mvn package \
  -Dwebclient.war=../oie-web-client/web-administrator/dist/oie-webadmin.war

# target/websupport-<version>.zip
```

### 签名构建（YubiKey）

需要持有代码签名证书的 YubiKey 和 OpenSC PKCS#11 库。一次性设置：将 `yubikey-pkcs11.cfg.example`
复制到 `yubikey-pkcs11.cfg` 并根据系统调整库路径，将证书链放在仓库根目录的 `certchain.pem` 中
（两个文件都不纳入版本控制）。

```bash
OIE_HOME=/path/to/oie mvn clean package -Psigning \
  -Dsigning.storepass=<yubikey-pin> \
  -Dwebclient.war=/path/to/oie-webadmin.war
```

省略 `-Dsigning.storepass` 可从 `YUBIKEY_PIN` 环境变量读取 PIN。
仅对 `websupport-shared.jar` 签名；WAR 保持与 Web 客户端发布资产字节相同，
以便发布工作流的摘要检查仍然有效。使用 `jarsigner -verify target/websupport-shared.jar` 验证。

Web 客户端发布工作流发布 `oie-webadmin.war`。Web 管理支持发布工作流下载 `pom.xml` 中
`webclient.version` 选择的发布版本，验证它，并在发布说明中记录确切的客户端标签和 SHA-256。
在发布线路匹配时保持该属性与 Web 管理支持版本相等，或在项目独立版本时更改它。
没有源代码检出、任意提交引用或需要维护的仓库变量。

工作流会将确切的已签名 ZIP、校验和与来源回执保留为 `signed-release-assets`，
然后在发布前先上传到草稿。如果发布或目录更新失败，重新运行失败的作业即可复用这些字节。
现有资产必须完全匹配；重新构建或新的签名时间戳不能替换已发布的包。
GitHub `starter` 上传为空时，需要先按错误提示清理特定的草稿资产，然后再重试。

## 说明

- 词汇描述使用内置映射或已启用扩展的清单声明，在运行时加载 SHARED 类，无编译时耦合。
- 静态资产使用 Servlet 容器的 MIME 表提供服务（`ServletContext.getMimeType`），
  对 `.js`/`.mjs` 固定为 `text/javascript; charset=utf-8`，确保 ES 模块始终可执行。
- 文件服务限制在每个扩展的 `webadmin/` 文件夹内（规范化路径包含；遍历和符号链接转义被拒绝）。
- Web 管理支持在支持的地方使用暂存文件和原子替换安装其 WAR。只读的 `webapps/` 目录会阻止 WAR 部署；
  API 仍然加载，因此引擎可以启动，复制失败会被记录。

## 许可证

MPL-2.0
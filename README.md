# OIE Web 管理支持

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
| `GET /webplugins` | 列出已安装、已启用且附带 Web UI 半部分的扩展（`webadmin/plugin.json`）。 |
| `GET /webplugins/{extension}/{path}` | 从扩展的 `webadmin/` 文件夹提供静态文件，使插件的浏览器 UI 跟随其安装的引擎。 |

Web 管理器在会话开始时探测这些端点——优先使用引擎原生端点（内置了这些端点的引擎），
然后回退到此插件——并在两者都不存在时优雅降级（无消息树、无服务器端验证、无引擎提供的插件 UI）。

## 安装——请先执行此操作

此插件是唯一无法通过社区商店自行安装的插件，因为商店自身的 UI 就是由它提供的。
引导顺序：

1. 从 [Releases](../../releases) 下载 `websupport-<version>.zip`。
2. 在 Web 管理器中：**扩展 → 安装**（扩展安装仅使用引擎核心 REST API，因此无需此插件即可工作）。
   经典 Swing 管理器同样可以。
3. 重启引擎。

之后，插件 UI（包括社区商店）将亮起，此插件会像其他扩展一样通过商店更新自身。

## 构建

需要 OIE/Mirth Connect 安装（或已构建的引擎树）以获取引擎 jar 包，以及已构建的 Web 客户端 WAR。
WAR 是必需的：

```bash
cd ../oie-web-client
npm ci
npm run build:war

cd ../oie-web-support-plugin
OIE_HOME=/path/to/oie mvn package   -Dwebclient.war=../oie-web-client/web-administrator/dist/oie-webadmin.war

# target/websupport-<version>.zip
```

### 签名构建（YubiKey）

需要持有代码签名证书的 YubiKey 和 OpenSC PKCS#11 库。一次性设置：将 `yubikey-pkcs11.cfg.example`
复制到 `yubikey-pkcs11.cfg` 并根据系统调整库路径，将证书链放在仓库根目录的 `certchain.pem` 中
（两个文件都不纳入版本控制）。

```bash
OIE_HOME=/path/to/oie mvn clean package -Psigning   -Dsigning.storepass=<yubikey-pin>   -Dwebclient.war=/path/to/oie-webadmin.war
```

省略 `-Dsigning.storepass` 可从 `YUBIKEY_PIN` 环境变量读取 PIN。
仅对 `websupport-shared.jar` 签名；WAR 保持与 Web 客户端发布资产字节相同，
以便发布工作流的摘要检查仍然有效。使用 `jarsigner -verify target/websupport-shared.jar` 验证。

Web 客户端发布工作流发布 `oie-webadmin.war`。Web 管理支持发布工作流下载 `pom.xml` 中
`webclient.version` 选择的发布版本，验证它，并在发布说明中记录确切的客户端标签和 SHA-256。
在发布线路匹配时保持该属性与 Web 管理支持版本相等，或在项目独立版本时更改它。
没有源代码检出、任意提交引用或需要维护的仓库变量。

## 说明

- 词汇描述（HL7 v2、X12、NCPDP、DICOM）在运行时从引擎自身的数据类型类按名称解析——
  无编译时耦合，未安装的数据类型只是不贡献描述。
- 静态资产使用 Servlet 容器的 MIME 表提供服务（`ServletContext.getMimeType`），
  对 `.js`/`.mjs` 固定为 `text/javascript; charset=utf-8`，确保 ES 模块始终可执行。
- 文件服务限制在每个扩展的 `webadmin/` 文件夹内（规范化路径包含；遍历和符号链接转义被拒绝）。
- Web 管理支持在支持的地方使用暂存文件和原子替换安装其 WAR。只读的 `webapps/` 目录会阻止 WAR 部署；
  API 仍然加载，因此引擎可以启动，复制失败会被记录。

## 许可证

MPL-2.0

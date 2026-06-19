# Xior-租房脚本
仅作提醒用的Xior租房脚本，要求有python环境；用codex辅助或自行安装，查找到房源时，会弹窗邮件提醒；实际抢房需要自己操作，脚本仅作提醒，不能全程代办（考虑封号风险）
----
用途
----
这个脚本用于监控 Xior Student Housing 在 Groningen 的公开房源接口和官方页面信号。
它只做检查和提醒，不会绕过 Cloudflare / Turnstile，不会自动登录，不会自动提交申请，不会签约或付款。
需要监控其他地区，可自行让Codex修改相关设置
脚本分Win、Mac版，各自下载

默认监控
--------
- Zernike Tower
- Zernike Tower - Short Stay
- Eendrachtskade
- Oosterhamrikkade

重要提醒
--------
- 真正可行动的结果必须来自 API 返回的具体 apartmentId / unit 信息。
- 官方页面上的 "rooms available"、"select your room" 只算线索，不能保证真的有房。
- 付款或提交申请前，必须人工核对楼栋、房型、日期、租金和个人信息。
- 如果页面显示 UnitId=0、今天日期、错误楼栋、房型不可用或付款前信息不一致，先停止。
- 不要把检查频率改得过高；网站出现 Too many requests 时脚本会自动退避。

运行环境
--------
- Windows 10/11
- Python 3.10 或更高版本
- Windows 自带 curl.exe 可用
- 可选：Outlook 或默认邮件客户端，用于 mailto 邮件草稿

快速开始：普通 Windows 直接安装
------------------------------
1. 解压 zip 到一个固定文件夹，例如：
   C:\Tools\xior-groningen-monitor

2. 推荐先运行配置向导：
   powershell -ExecutionPolicy Bypass -File .\setup-config-wizard.ps1

   配置向导会依次询问：
   - Windows 计划任务名称
   - 提醒邮箱
   - 监控开始日期
   - 重点监控阶段结束日期
   - 中等强度阶段结束日期
   - 最后监控日期
   - 到期自动删除计划任务的日期
   - 要监控哪些楼栋，以及它们的优先级
   - 是否弹窗、是否打开官方网页、是否复制申请链接、是否打开邮件草稿
   - 是否保留请求节流
   - 是否自定义每轮 API/页面检查数量、不同优先级的实际请求间隔
   - 是否在自动删除前生成脱敏留档 zip

3. 如果不想用向导，也可以直接打开 config.json，至少修改这些字段：
   - recipients: 改成你自己的邮箱地址
   - delete_task_on_or_after_local: 到期自动删除计划任务的日期

4. 测试脚本：
   在 PowerShell 中进入解压目录，运行：
   python .\XiorZernikeMonitor.py --dry-run --no-popup --no-open

5. 安装 Windows 计划任务：
   右键 PowerShell，普通权限通常即可，然后进入解压目录运行：
   powershell -ExecutionPolicy Bypass -File .\install-windows-task.ps1

6. 查看日志：
   monitor.log

7. 卸载计划任务：
   powershell -ExecutionPolicy Bypass -File .\uninstall-windows-task.ps1

快速开始：交给 Codex 安装
-------------------------
1. 解压 zip 到一个固定文件夹。
2. 打开 codex-install-prompt.txt，把里面的提示词发给 Codex，并把解压后的文件夹路径填进去。
3. Codex 应先读取 codex-install-instructions.md，再调整 config.json、运行验证，最后安装计划任务。
4. 这种方式适合需要精调参数的情况，例如：
   - 只查 Zernike Tower 和 Short Stay
   - 已经租到房，只想低频等待更好的房
   - 6 月高频、7 月中频、8 月低频
   - 每轮只查 1 个 API 房型，页面检查间隔拉长，避免触发限流

配置说明
--------
config.json 中比较重要的字段：

- start_date_local
  监控开始日期。日期格式必须是 YYYY-MM-DD。

- delete_task_on_or_after_local
  到达这个日期后，脚本会尝试删除自己的 Windows 计划任务，并清理本地提醒辅助文件。
  通常建议把它设为“最后监控日期的第二天”。

- date_check_periods
  分阶段运行频率。每一段包含 start、end、interval_minutes。
  例如：6 月 1 分钟唤醒一次，7 月 5 分钟一次，8 月 15 分钟一次。
  注意：脚本即使频繁唤醒，也会再通过 request_throttle 控制实际请求量。

- temporary_overrides
  临时覆盖规则。适合“今天先消停一天，但如果 Zernike 有房仍提醒”这种场景。
  可以设置某一天的 interval_minutes，并通过 enabled_property_ids 临时只监控某些楼栋。
  过了 end 日期会自动失效，不需要手动恢复长期配置。

- properties
  每个楼栋的配置。enabled 控制是否监控；priority 控制优先级，1 最高，3 较低。
  默认包含 Zernike Tower、Zernike Tower - Short Stay、Eendrachtskade、Oosterhamrikkade。

- recipients
  有房时打开 mailto 草稿的收件人列表。

- smtp.enabled
  默认为 false。false 时只打开邮件草稿；true 时需要配置 SMTP 和环境变量 XIOR_SMTP_PASSWORD。

- request_throttle
  请求节流配置。脚本可以每分钟醒来，但每轮只检查少量 API/页面，降低触发限流的概率。
  重点字段：
  - max_api_room_checks_per_run: 每轮最多检查几个 API 房型
  - max_page_checks_per_run: 每轮最多检查几个房源页面
  - api_interval_minutes_by_priority: 不同优先级 API 的最小检查间隔
  - page_interval_minutes_by_priority: 不同优先级页面的最小检查间隔

- rate_limit_backoff_minutes / max_rate_limit_backoff_minutes
  遇到 Too many requests 后的退避窗口。连续限流会逐步延长。

- archive_before_delete
  到达 delete_task_on_or_after_local 后，脚本删除计划任务前先生成一个脱敏留档 zip。
  默认输出到 Downloads，并排除 monitor.log、state.json、last-*.html、备份和缓存。

- safe_automation
  提醒动作开关：
  - open_helper_page_on_alert: 打开本地快速行动页
  - open_application_launcher_on_alert: 打开本地申请链接页
  - open_official_page_on_alert: 打开官方房源页
  - copy_first_link_to_clipboard_on_alert: 复制第一个申请链接
  - open_mailto_on_alert: SMTP 未配置时打开邮件草稿
  - show_popup_on_alert: 显示 Windows 弹窗

提醒方式
--------
当 API 返回真实房源时，脚本会：
- 弹出 Windows 提醒
- 发出系统提示音
- 打开本地快速行动页
- 打开官方页面
- 复制第一个申请链接到剪贴板
- 如果 SMTP 未配置，则打开 mailto 邮件草稿

安全边界
--------
这个脚本适合“提醒你去人工操作”，不适合也不会做“自动抢房”。
请尊重网站规则，避免高频强刷、绕过验证、自动提交申请或自动付款。

隐私说明
--------
这个共享包不应包含原作者个人信息。打包时已排除：
- 个人邮箱
- monitor.log
- state.json
- last-*.html 缓存页面
- Python __pycache__
- 本机 Python 绝对路径

config.json 里的默认邮箱是 your-email@example.com。分享前仍建议全文搜索一次自己的姓名、邮箱和本机路径。

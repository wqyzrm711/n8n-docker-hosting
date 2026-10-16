# n8n docker hosting：它真正的花费、哪些规格很重要，以及一套不会导致廉价 VPS 崩溃的设置

在您自己的 Docker 主机上运行 n8n 比大多数人预期的要便宜。n8n Cloud 的 Starter 套餐标价为每月 $20，包含 2,500 次执行，而 Pro 套餐为每月 $50。一台能够轻松运行自托管版本的 VPS 每月仅需 $2–4，且没有任何执行次数限制。

这个决定还有一个截止日期。n8n 的 changelog 显示，v3.0（预计 2026 年 10 月）将完全放弃安装支持，从该版本开始，Docker 将成为受支持的自托管方式。因此，真正的决定在于选择哪种服务器以及什么才是合理的价格。

本指南涵盖了人们在搜索 n8n Docker 托管时真正关心的三件事：n8n 在每个工作负载级别所需的规格、合理的托管费用（以及廉价提供商在何处隐藏了陷阱），以及防止小型 VPS 在运行中途 OOM-kill 您的工作流的两个设置。

> **快速选择：** RackNerd 的年度 KVM 特价套餐 1 GB VPS 起价为**每年 $10.99**。3 GB 套餐（$33.88/年，2 vCPU）以每月不到 $3 的价格接近 n8n 推荐的生产规格，而 4 GB 套餐（$43.88/年）则完全符合该规格，并留有余地。这是非托管托管，因此您需要自己进行设置和备份。👉 [查看 RackNerd 当前的 KVM 特价](https://bit.ly/my-rn)

## n8n 实际需要什么样的 Docker 主机

n8n 自身的指南指出，可靠的消费底线是 2 vCPU 和 4 GB RAM，外加 20–40 GB 的 SSD。低于该配置软件仍然可以运行：1 vCPU / 2 GB 的机器可以很好地处理个人自动化。您只是用余量换取了更低的账单，其余部分展示了这种权衡何时会适得其反。

| 工作负载 | vCPU | RAM | 存储 | 运行内容 |
| --- | --- | --- | --- | --- |
| 轻量 / 试用 | 1 | 2 GB | 10–20 GB | 个人工作流、轻量 Webhook |
| 生产环境（推荐） | 2+ | 4 GB | 20–40 GB | n8n + PostgreSQL，并发执行 |
| 队列模式 | 2+ | 4–8 GB | 40 GB+ | 增加 Redis 和 Postgres 容器 |
| AI Agent 重度使用 | 4+ | 8 GB | 50 GB+ | 工具调用在 RAM 中持有上下文 |

队列模式是让人们感到意外的地方。在处理大量 Webhook 接收时，意味着三个容器——n8n、PostgreSQL 和 Redis——在同一台机器上，除非您将它们拆分。请为这三者都预留尺寸。

AI Agent 工作流是另一个内存消耗大户：在运行期间，每个工具调用都会将上下文保留在 RAM 中。一旦引入 Agent，“4 GB 绰绰有余”的机器很快就变得不够用了。

## 为什么 Docker 即将成为自托管 n8n 的唯一途径

因为 npm 安装方式正在被淘汰。根据 n8n 的 changelog，v3.0（预计 2026 年 10 月）移除了 npm 安装支持。从那时起，每个自托管实例都在容器中运行。

Docker 路径本身具有实际优势。compose 文件存放在 git 中，因此您的配置是受版本控制的。固定镜像标签意味着升级需要刻意的 `docker compose pull`，回滚也同样需要刻意操作。持久化卷保存了所有重要内容，这使得主机之间的迁移变成了一项复制工作。

关于硬件的一点说明：Docker 需要基于真正虚拟化的 VPS。基于 KVM 的方案——这也是 RackNerd 所销售的——表现得像一台小型专用服务器，并且可以毫无障碍地运行 Docker。较旧的基于容器的虚拟化（如 OpenVZ）与现代 Docker 存在冲突；即使价格看起来很诱人，也要跳过它。

## 自托管 n8n 服务器应该花费多少

在年度预算定价下，每月 $2–4 就能买到符合 n8n 生产推荐的硬件。以下是相同或接近规格下的常见选择：

| 主机 | 套餐 | 规格 | 价格 |
| --- | --- | --- | --- |
| RackNerd | KVM 特价，3 GB | 2 vCPU / 3 GB / 50 GB SSD | $33.88/年 (~$2.82/月) |
| RackNerd | KVM 特价，4 GB | 2 vCPU / 4 GB / 65 GB SSD | $43.88/年 (~$3.66/月) |
| Hetzner | CAX11 (ARM) | 2 vCPU / 4 GB | €3.29/月 |
| DigitalOcean | Basic Droplet | 2 vCPU / 4 GB | $24/月 |
| Contabo | VPS S | 4 vCPU / 8 GB / 200 GB | ~$7/月 |
| n8n Cloud | Starter | 托管，2,500 次执行/月 | $20/月 |

AWS 是一个警示故事，所以它值得单独一提。t3.medium（2 vCPU / 4 GB）仅计算费用就标价为每月 $30.37，外加约 $6.40 的存储费用。流量费才是账单失控的地方：超过最初的 100 GB 后，AWS 每增加 1 GB 收费 $0.09，因此推送 1 TB 流量每月会增加约 $81。返回 Webhook 负载或代理 API 响应的 n8n 实例可以轻松达到这种流量。Hetzner、DigitalOcean、Contabo 和 RackNerd 都捆绑了以 TB 为单位的流量，而不是按 GB 计费。

与 n8n Cloud 的盈亏平衡计算变得非常简单：以 $10.99/年的价格计算，一个月的 Starter 套餐费用比 RackNerd 最小的 VPS 一整年的费用还要高。DigitalOcean 的 4 GB 套餐每年运行成本为 $288；RackNerd 4 GB 套餐为 $43.88，大约是前者的 15%。按执行次数限制的 SaaS 定价会随着您的使用量而增加；固定价格的 VPS 则不会。 👉 [对比 RackNerd 的年度定价](https://bit.ly/my-rn)

## RackNerd 套餐，映射到 n8n 工作负载

简短的回答：$10.99–$85.88/年 涵盖了从个人沙盒到 8 GB 队列模式服务器的所有需求。RackNerd 销售非托管 KVM VPS，其声誉建立在年度特价定价之上，这大大低于大多数按月计费的竞争对手。当前的 lineup：

| 套餐 | vCPU | RAM | 存储 / 流量 | 年度价格 | 订购 |
| --- | --- | --- | --- | --- | --- |
| 1 GB | 1 | 1 GB | 20 GB / 1 TB | $10.99/yr | [ 订购 1 GB 套餐](https://bit.ly/my-rn) |
| 2 GB | 1 | 2 GB | 30 GB / 2.5 TB | $21.99/yr | [ 订购 2 GB 套餐](https://bit.ly/my-rn) |
| 3 GB | 2 | 3 GB | 50 GB / 4 TB | $33.88/yr | [ 订购 3 GB 套餐](https://bit.ly/my-rn) |
| 4 GB | 2 | 4 GB | 65 GB / 5 TB | $43.88/yr | [ 订购 4 GB 套餐](https://bit.ly/my-rn) |
| 8 GB | 4 | 8 GB | 120 GB / 6 TB | $85.88/yr | [ 订购 8 GB 套餐](https://bit.ly/my-rn) |

订购前，有一些事实：

- 流量包含在 1 Gbps 端口的每月 TB 数中。典型的 n8n 流量——Webhook 调用、API 请求、数据库写入——即使对于 1 TB 套餐也仅使用极小部分。只有在代理大文件时它才变得重要，即便如此，您的空间也比免费提供的 AWS 要多。
- 位置因套餐而异。大多数特价套餐覆盖多个美国数据中心（洛杉矶、圣何塞、达拉斯、芝加哥、纽约、亚特兰大、西雅图）以及阿姆斯特丹，但确切列表会在结账时显示。选择最接近您工作流调用的 API 的位置。
- 套餐是非托管的：完全的 root 访问权限，所有 OS 级别的维护都由您自己完成。支持部门处理的是基础设施——网络、硬件、虚拟化层。
- 定价是促销性质的，并在节假日前后有所变动，部分活动中会出现中间档位（1.5 GB、2.5 GB、5 GB）。订购页面总是显示实时的 lineup。

对比 Hetzner，存在一个架构上的差异：RackNerd 的廉价套餐是 x86 架构，而 Hetzner 的性价比之王（CAX11）是 ARM 架构。n8n 的官方镜像两者都支持。如果您计划添加社区节点或自定义 Docker 镜像，x86 架构消除了一整类构建上的麻烦。

## 保持廉价 VPS 存活的两个设置

小型 VPS 套餐在 n8n 中以特定方式失败：工作流在运行中途死掉，日志显示进程被杀掉，看起来像是 n8n 的 bug。这是内存管理的问题，两个环境变量可以解决它。

**二进制数据默认保存在 RAM 中。** 除非您设置了 `N8N_DEFAULT_BINARY_DATA_MODE=filesystem`，否则 n8n 会将文件负载保存在内存中。在 1–2 GB 的机器上，一个移动附件的工作流会 OOM-kill 自己。设置该变量，负载就会转移到磁盘上。

Node.js 堆会忽略容器限制。V8 根据主机总 RAM 来确定其堆大小，而不是根据您的 Docker 内存上限，因此在容器中它可能会超过限制，直到有东西将其杀掉。显式地限制它：


NODE_OPTIONS=--max-old-space-size=2048


为 OS 和 Postgres 留出 512 MB–1 GB 空间。在 3–4 GB 的档位上，2048–3072 是一个合理的范围。

添加一个 swapfile 作为最后的缓冲区。这只需要两分钟的工作，就能将 OOM 峰值转化为减速，而不是崩溃：


sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab


## 设置：从裸 VPS 到可用的 HTTPS 约 1 小时

有 Linux 服务器经验？总共预算 30–60 分钟。步骤如下：

1. **订购 VPS**，选择 Ubuntu 24.04 LTS，添加 SSH 密钥，并记录 IP。配置通常是自动的，需要几分钟。
2. **锁定防火墙**，只留三个端口：22 (SSH)、80 (HTTP，重定向到 HTTPS)、443 (HTTPS)。其他所有端口保持关闭，n8n 本身绝不应能直接访问。
3. **安装 Docker** 加上 compose 插件（Docker 的官方安装脚本两者都处理），然后将您的用户添加到 docker 组。
4. **部署堆栈。** 一个最小化的生产环境 compose 将 n8n 与 PostgreSQL 配对：

yaml
services:
  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: n8n
      POSTGRES_USER: n8n
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - pg_data:/var/lib/postgresql/data

  n8n:
    image: docker.n8n.io/n8nio/n8n:latest  # pin a version tag in production
    restart: unless-stopped
    ports:
      - "127.0.0.1:5678:5678"
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      - N8N_HOST=automation.yourdomain.com
      - WEBHOOK_URL=https://automation.yourdomain.com/
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_DEFAULT_BINARY_DATA_MODE=filesystem
      - GENERIC_TIMEZONE=America/New_York
      - TZ=America/New_York
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      - postgres

volumes:
  pg_data:
  n8n_data:


注意端口绑定：`127.0.0.1:5678` 意味着 n8n 只能通过反向代理访问，绝不能直接从互联网访问。

5. **添加用于 HTTPS 的反向代理。** Caddy 用两行配置就能做到这一点，并自动处理 Let's Encrypt 证书：


automation.yourdomain.com {
    reverse_proxy n8n:5678
}


首先将 A 记录指向您的服务器，然后 Caddy 会自行整理证书。如果 Nginx 加 certbot 已经是您的习惯，它的工作方式也相同。

四个环境变量决定了设置是否能经受住现实的考验：

- `N8N_ENCRYPTION_KEY` 加密每个存储的凭据。**丢失它，n8n 中保存的每个登录信息都将无法恢复。** 在首次启动前生成一个，并在同一分钟内将副本放入您的密码管理器。
- `WEBHOOK_URL` 告诉 n8n 它的公共地址。跳过它，Webhook URL 将针对内部端口生成，导致经典的静默失败：工作流在编辑器中“在测试中工作”，然后在生产环境中从不触发。
- `N8N_DEFAULT_BINARY_DATA_MODE=filesystem`，如上所述。
- `GENERIC_TIMEZONE` 和 `TZ` 保持调度触发器在您的本地时间触发，而不是 UTC。

关于数据库选择：SQLite（n8n 的默认值）在您超过每月几千次执行之前都足够了。仅启动 SQLite 意味着从 compose 文件中删除 postgres 服务。当您超出其能力范围时，迁移是一项导出和恢复的工作——规划好窗口期，不要在生产环境实时进行。

在非托管 VPS 上，备份是您自己的工作。`n8n_data` 卷包含 SQLite 数据库和加密密钥，使其成为需要复制的最重要内容；对于 Postgres，安排 `pg_dump` 到异地存储。在您需要恢复之前设置好 cron，并在风险较低时测试一次恢复。

如果您更愿意在一个临时机器上排练，最小的档位足够便宜，可以当作沙盒：👉 [以每年 $10.99 的价格开始使用 1 GB 套餐](https://bit.ly/my-rn)

## 何时 RackNerd 是错误的选择

诚实的限制，因为它们确实存在。RackNerd 是非托管的，所以更新、SSL、备份和安全加固都由您自己负责——如果那句话让您感到疲惫而不是兴奋，那么托管选项更适合：n8n Cloud（Starter $20/月，Pro $50/月）或为您运行容器的托管 n8n 主机（在当前提供商中，价格从每月约 $3–9 不等）。

预算型 KVM 也意味着共享 CPU。对于 cron 任务和 Webhook 胶水代码，没人会注意到。对于持续的重负载，专用 CPU 套餐（Hetzner 的专用线路起价约为每月 €14）在压力下预测性更好。

合规团队也会改变计算方式。如果您需要 SOC 2 认证、官方 SLA 和欧盟数据居留权相关文书，n8n Cloud 正是为这种对话而构建的选项。

## FAQ

**n8n 可以在 1 GB 的 VPS 上运行吗？**
可以，用于轻量个人使用。SQLite，启用 swap，二进制数据存放在文件系统上，它就可以运行。对于并发执行或附件繁重的工作流，预期会有困难——2 GB 档位是舒适的底线。

**促销价格是否以相同的费率续订？**
RackNerd 历史上一直以注册时的费率续订年度特价。请查看结账时显示的条款；那才是具有约束力的版本。

**廉价的 VPS 对于 Webhook 来说足够可靠吗？**
Webhook 触发器需要一个始终运行的进程和一个带有 SSL 的稳定公共 URL。VPS 提供了这两者。实际破坏调度触发器和队列模式的平台是那些杀掉空闲容器的无服务器平台——KVM VPS 没有这种行为。

**我需要队列模式吗？**
不是立刻需要。带有 PostgreSQL 的单实例 n8n 处理大多数小型企业工作负载。当 Webhook 量变得足够大导致接入停滞时，队列模式就显得很重要——它增加了 Redis，并将 RAM 需求推高至总共 4–8 GB。

**服务器应该位于何处？**
靠近您的工作流所交互的服务，并且位于您的数据规则所要求的区域内。阿姆斯特丹覆盖了提供该选项的套餐的欧盟居留权要求；除此之外，美国各位置主要在您调用的 API 的延迟上有所不同。

# sing-box-subscribe 生成 sing-box 配置
{docsify-updated}

在 macOS 系统上，当通过 Homebrew 安装 Python 3.12 或更高版本后，直接使用 pip3 install 安装第三方库会触发 `externally-managed-environment` 错误。这是因为系统出于稳定性考虑，禁止对全局 Python 环境进行直接修改。为安全、隔离地安装不在 Homebrew 管理范围内的 Python 库，推荐使用虚拟环境.

```
gcl https://github.com/PerfecterDay/sing-box-subscribe.git

python3 -m venv ~/python/venv 
source ~/python/venv/bin/activate
pip3 install -r requirements.txt 

修改 providers.json 中的 subscribes[0].url 为订阅地址，其他配置也可以根据需要修改

python main.py 选择需要的模板，然后会生成 config.json 文件
```
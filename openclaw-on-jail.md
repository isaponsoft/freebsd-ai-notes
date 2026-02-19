## Host

```bash
bastille create openclaw 15.0-RELEASE Your_Jail_IP_Address Your_Bridge_Interface
```

## jail

```bash
root# bastille console openclaw
root# pkg update
root# pkg install npm git cmake
# openclaw user
root# adduser openclaw-user
root# su openclaw-user
```
## install

```bash
openclaw-user% INSTALL_ROOT=~/.local
openclaw-user% cd ~
openclaw-user% mkdir $INSTALL_ROOT
openclaw-user% npm install -g openclaw --prefix $INSTALL_ROOT
```

## run

```bash
openclaw-user% $INSTALL_ROOT/bin/openclaw
openclaw-user% $INSTALL_ROOT/bin/openclaw setup
openclaw-user% cat $INSTALL_ROOT/bin/.openclaw/openclaw.conf
```

### before

```json
{
  "agents": {
    "defaults": {
      "workspace": "/home/openclaw-user/.openclaw/workspace"
    }
  },
  "commands": {
    "native": "auto",
    "nativeSkills": "auto"
  },
  "meta": {
    "lastTouchedVersion": "2026.2.17",
    "lastTouchedAt": "2026-02-18T10:44:44.154Z"
  }
}
```

### after

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "llama/qwen2.5",
      },
      "workspace": "/home/openclaw-user/.openclaw/workspace"
    }
  },
  "commands": {
    "native": "auto",
    "nativeSkills": "auto"
  },
  "meta": {
    "lastTouchedVersion": "2026.2.17",
    "lastTouchedAt": "2026-02-18T10:44:44.154Z"
  },
  "models": {
     "providers": {
        "YourProvidorName": {
          "baseUrl": "http://YourLocalLLM-Server/v1",
          "apiKey": "Dummy.",
          "api": "openai-completions",
          "models": [
            {
              "id": "UseModelName",
              "name": "Your Management name",
              "input": [ "text" ],
              "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
              "contextWindow": 16384,
              "maxTokens": 4096
		    }
          ]
        }
     }
  },
  "gateway": {
    "port": 12345,
    "mode": "local",
    "bind": "lan",
    "auth": {
      "mode": "token",
      "token": "YourAuthToken"
    }
  }
}
```

```bash
openclaw-user% .local/bin/openclaw gateway

🦞 OpenClaw 2026.2.17 (4134875) — I don't just autocomplete—I auto-commit (emotionally), then ask you to review (logically).

10:58:43 [canvas] host mounted at http://0.0.0.0:12345/__openclaw__/canvas/ (root /home/openclaw-user/.openclaw/canvas)
10:58:43 [heartbeat] started
10:58:43 [health-monitor] started (interval: 300s, grace: 60s)
10:58:43 [gateway] agent model: llama/qwen2.5
10:58:43 [gateway] listening on ws://0.0.0.0:12345 (PID 44615)
10:58:43 [gateway] log file: /tmp/openclaw/openclaw-2026-02-18.log
10:58:43 [browser/service] Browser control service ready (profiles=2)
```
## Install reverse proxy

```bash
root# pkg install caddy
root# vi 
```

```json
Your_Jail_IP_Address {
                reverse_proxy 127.0.0.1:12345
                log {
                                output file /var/log/caddy/access.log
                                format json
                }
}
```

## Check

Brower open : https://Your_Jail_IP_Address/

次のようなメッセージが表示されるはず。`~/.openclaw/openclaw.conf` の `gateway.auth.token` の値を `Overview` → `Gateway Token` に入力

```disconnected (1008): unauthorized: gateway token missing (open the dashboard URL and paste the token in Control UI settings)```


## Pairing

### ~/.openclaw/devices/pending.json

`silent` modify `true`

```json
{
  "48401312-6b54-457d-aecd-a950422f5482": {
    "requestId": "AAAAAAAAAAAAAAAAAAAAAAAAAAAA",
    "deviceId": "BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB",
    "publicKey": "CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC",
    "platform": "Win32",
    "clientId": "openclaw-control-ui",
    "clientMode": "webchat",
    "role": "operator",
    "roles": [
      "operator"
    ],
    "scopes": [
      "operator.admin",
      "operator.approvals",
      "operator.pairing"
    ],
    "silent": true,
    "isRepair": false,
    "ts": 1771413305182
  }
}

```

### llama-server

```bash
llama-server --ctx-size 32768 --host 0.0.0.0 --port 11434 --chat-template chatml -m "Model file path"                                         
```

- `--ctx-size` はOpenClawが32768以上を推奨している。一応16384でもギリギリ動く。少なくともOpenClaw側のmodelに指定した`contextWindow`よりも大きくすること。
- `--chat-template chatml` これが最も重要。これが無いと、OpenClawに指示を出したときに永遠に返事が返ってこない事がある。本来であれば、llama-server に入っている jinja がOpenClawからの命令を自動解釈してくれるのだが、それがうまく働かないことがある。そこで明示的に `chatml` を指定することでやりとりが安定する。将来的には自動検出が上手くいって不要になるかもしれない。

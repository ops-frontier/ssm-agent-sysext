# ssm-agent-sysext
SSM Agent を Flatcar Linux で利用するための sysext

## 全体像
systemd-sysext は、指定したディレクトリ構造（イメージ）を /usr などのディレクトリに overlayfs でマージする仕組みです。そのため、以下のようなディレクトリツリーを作成してイメージ化（SquashFS）します。
```
amazon-ssm-agent-ext/
├── usr/
│   ├── bin/
│   │   └── amazon-ssm-agent                 <-- バイナリ本体（Actions でバージョンを指定してダウンロード）
│   └── lib/
│       ├── extension-release.d/
│       │   └── extension-release.amazon-ssm-agent <-- OS一致検証用のメタデータ
│       └── systemd/
│           └── system/
│               └── amazon-ssm-agent.service <-- systemd サービス定義ファイル
└── etc/
    └── amazon/
        └── ssm/
             └── ssm-activation.env    <-- 登録用環境変数(Butaneなどで設定)
```

## Actions でのリリース

systemd-extention は当該リポジトリの Github Releases にリリースされる。Actions で amazon-ssm-agent の VERSION をパラメータとして以下の手順で当該リポジトリにリリースされる

1. `https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/${VERSION}/linux_amd64/amazon-ssm-agent`を `amazon-ssm-agent-ext/usr/bin/amazon-ssm-agent` にダウンロードする
2. `mksquashfs amazon-ssm-agent-ext/ amazon-ssm-agent.raw -noappend` を実行して、`amazon-ssm-agent.raw`を作成する
3. `amazon-ssm-agent.raw` を Github の Releases に  ${VERSISON} という名前でリリースされる。常に latest タグが付与される

## Butane サンプル

Flatcar Container Linux にインストールする際の Butane YAML ファイルのサンプルを示す。
amazon-ssm-agent を動作させるには最初にアクティベーションコードで登録する必要がある。この登録処理は ssm-register.service によって実行する。

```
variant: flatcar
version: 1.1.0
storage:
  files:
    # 1. sysext イメージのダウンロード
    - path: /etc/extensions/amazon-ssm-agent.raw
      mode: 0644
      contents:
        source: https://github.com/ops-frontier/ssm-agent-sysext/releases/download/latest/amazon-ssm-agent.raw

    # 2. 登録用のアクティベーションコード、リージョンなどをホスト側に配置
    - path: /etc/amazon/ssm/ssm-activation.env
      mode: 0600
      contents:
        inline: |
          SSM_ACTIVATION_CODE=アクティベーションコード
          SSM_ACTIVATION_ID=アクティベーションID
          AWS_REGION=AWSリージョン

  links:
    # 3. sysext 側のサービスを multi-user.target で起動させるシンボリックリンク
    #    ssm-register.service, amazon-ssm-agent.service は sysext イメージ内にあるため Ignition 処理時点では
    #    まだ存在しない。enabled: true は unit ファイルを参照するためエラーになる
    #    可能性があるので、links で直接シンボリックリンクを作成する。
    - path: /etc/systemd/system/multi-user.target.wants/ssm-register.service
      target: /usr/lib/systemd/system/ssm-register.service
      hard: false
    - path: /etc/systemd/system/multi-user.target.wants/amazon-ssm-agent.service
      target: /usr/lib/systemd/system/amazon-ssm-agent.service
      hard: false

systemd:
  units:
    # systemd-sysext自体の有効化
    - name: systemd-sysext.service
      enabled: true
```


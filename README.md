# working-skills

作業用スキルのプラグインマーケットプレイス。

このリポジトリは Claude Code のプラグインマーケットプレイスとして構成されている。
現在は次の2つのプラグインを提供する（スキルは今後追加していく）。

| プラグイン | 用途 | パス |
| --- | --- | --- |
| `develop` | コーディング開発向けのスキル集 | `plugins/develop` |
| `daily` | 日常業務向けのスキル集 | `plugins/daily` |

## インストール方法

Claude Code 上で以下を実行する。

```
# マーケットプレイスを登録（GitHub リポジトリを指定）
/plugin marketplace add haya790315/working-skill

# プラグインをインストール
/plugin install develop@powerful-skills
/plugin install daily@powerful-skills
```

`owner/repo` 形式を指定すると、GitHub 上の
`.claude-plugin/marketplace.json` を読み込んでマーケットプレイスを登録する。
ローカルパスや完全な Git URL（`https://github.com/haya790315/working-skill.git`）でも登録できる。

## スキルの追加方法

各プラグインの `skills/` ディレクトリ配下にスキルを配置すると、
インストール時に自動で認識される。

```
plugins/
├── develop/
│   ├── .claude-plugin/plugin.json
│   └── skills/
│       └── <skill-name>/
│           └── SKILL.md
└── daily/
    ├── .claude-plugin/plugin.json
    └── skills/
        └── <skill-name>/
            └── SKILL.md
```

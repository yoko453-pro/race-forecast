# レース予報 公開サイト

生成済みの予報HTMLをそのまま置くだけで、トップページ・開催日ページ・競馬場ページが自動生成される静的サイト。
ビルドは標準ライブラリのみ、出力も静的ファイルのみなので GitHub Pages / Cloudflare Pages / Netlify のいずれにもそのまま載る。

## フォルダ構成

```
build_index.py                  ビルドスクリプト
templates/index.template.html   一覧ページのテンプレート（デザインはここだけ触る）
public/                         公開ルート。ここをそのままデプロイする
├── index.html                  ← 生成物：全体トップ（開催日 → 競馬場 → レース）
├── data/index.json             ← 生成物：マニフェスト（外部利用・API代わり）
└── 2026-08-02/
    ├── index.html              ← 生成物：開催日トップ
    └── chukyo/
        ├── index.html          ← 生成物：競馬場トップ
        └── 04R/
            ├── index.html      ← 予報HTML本体（予報生成器の出力をそのまま配置）
            └── meta.json       ← 任意。抽出値を上書きしたいときだけ置く
```

- 日付ディレクトリは `YYYY-MM-DD` 固定。
- 競馬場ディレクトリはローマ字スラッグ（`chukyo`, `tokyo`, `hanshin` …）。URLにマルチバイト文字を入れないため。JRA10場＋地方主要場を `build_index.py` の `COURSE_SLUGS` に定義済み。
- レースディレクトリは2桁ゼロ埋め＋`R`（`04R`）。ソート順が崩れない。
- レースは必ず `.../04R/index.html`。`.../04R/` で開けるので、URLに `.html` が出ない。

## 使い方

```bash
# 1. 予報HTMLを規定の階層へ配置する（日付・競馬場・レース番号はHTMLから自動判定）
python build_index.py add forecast-2026080207020404-700784f3448740f0b5bd.html

# 判定できない場合は明示する
python build_index.py add forecast-xxxx.html --date 2026-08-02 --course 中京 --race 4

# 2. 走査して一覧ページ群とマニフェストを再生成する
python build_index.py build

# 3. ローカル確認
python -m http.server 8000 --directory public
```

`build` は毎回 `public/` 全体を走査して作り直す冪等な処理。予報を差し替えたら `build` を流すだけでよい。

## 予報HTMLから拾っている項目

`extract_meta()` が本体HTMLから以下を抽出する（正規表現ベース、依存なし）。

| 項目 | 抽出元 |
| --- | --- |
| 競馬場・レース番号 | `<title>中京4R レース予報</title>` |
| 更新段階・更新時刻・Forecast ID | `<p class="meta">` |
| 距離・頭数・馬場 | `data-section="RACE_OVERVIEW"` |
| まぎれ指数・警戒レベル | 総合まぎれ指数ゲージの `aria-valuenow` ／ 「警戒レベルは◯◯です」 |
| 確度レベル・確度スコア | `data-section="CONFIDENCE"` |
| 複勝率トップ・中心シナリオ | 該当セクションの文面 |
| オッズ有無・警告件数 | `data-section="DATA_WARNINGS"` |

抽出は本体HTMLの文面に依存するので、生成器側の文面を変えるときは `extract_meta()` も合わせて直すこと。
**より堅くしたい場合は、予報生成器の側で `meta.json` を同じディレクトリに吐くのが早い。** 存在すれば抽出値より優先される。

```json
{ "magire_score": 64.2, "magire_level": "HIGH", "confidence_level": "LOW", "odds_available": false }
```

## 一覧ページの構成

- **まぎれ帯** — 競馬場見出しの横に、その開催の全レースのまぎれ指数を棒で並べた小さなチャート。1日の荒れやすさの分布が一目で分かる。棒をクリックすると該当レース行へ飛ぶ。色は警戒レベル（HIGH=赤／MEDIUM=橙／LOW=緑）。
- **絞り込み** — 競馬場名・レース・注目馬・シナリオ名の部分一致検索と、警戒レベルのチップ絞り込み。
- **オッズ未取得マーク** — `◆` を付けて、odds-freeのみの予報だと分かるようにしてある。
- 配色・タイポは予報本体ページのトークン（`--turf #256b52` ほか）をそのまま引き継いでいるので、一覧から本体へ遷移しても地続きに見える。
- データは各ページに JSON として埋め込まれるため、`file://` で直接開いても動く。

## デプロイ

GitHub Pages なら `public/` を `docs/` にリネームするか、Actionsで `public` を artifact にする。

```yaml
# .github/workflows/pages.yml の要点
- run: python build_index.py build
- uses: actions/upload-pages-artifact@v3
  with: { path: public }
```

`public/index.html` には `<meta name="robots" content="noindex">` を入れてある。検索エンジンに載せてよい運用なら外す。

## 注記

まぎれ指数は結果分布の分散であって期待値ではない。一覧上で指数の高い順に並べたくなるが、それは「荒れやすい順」であって「買うべき順」ではない点は本体ページと同じ扱いにしてある。

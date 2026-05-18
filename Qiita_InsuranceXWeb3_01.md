# 【保険×Web3 #1】DeFi被害は数十億ドル、保険適用はごくわずか — 既存DeFi保険を担保する超過損害額再保険を概念実装する

## TL;DR

- Web3 業界のハッキング被害は単年で数十億ドル規模（Chainalysis：2024年 約$2.2B、2025年 約$3.4B）。**冒頭の主要6事例だけで累計 $400 億超**
- 一方、**DeFi 保険のカバー率は DeFi 全体 TVL の約 0.25%**（市場リーダー Nexus Mutual 基準・Lombard Notes 2026/1 分析）
- DeFi 文化は **保護より利回り・速度を優先**し、損失を実験のコストとして受容する傾向がある（Lombard Notes）。既存DeFi保険の普及は限定的
- 既存DeFi保険（Nexus Mutual 等）の支払い能力が事象規模を超えたとき、被害者は救済されない
- 本シリーズは、その「**保険会社のための保険 = 超過損害額再保険**」を Web3 に持ち込む概念実装 (仮称 Campanella) の出発点
- 第1回（今回）は前提・設計編。実装コードは Stage A 開始時に GitHub 公開予定（BUSL 1.1）
- **全文・思想・著者の背景・運営方針は note 全文版** でお読みいただけます（リンクは末尾）

---

## 本記事の前提

本記事は **AI 時代の書き方そのものへの実験** として書いている。

- **思想・問題設定・選別・校閲・校正**：cmalu ractu
- **文章・コード（概念実装スケッチ）・調査要約の生成**：Claude（Anthropic）

AI に文章とコードの生成、公開情報の調査要約を任せ、人間が思想と選別を引き受ける書き方が、これからの専門領域発信のひとつの形になり得るかを、本シリーズ全体で実地に試している。記事中の文章・コード・要約は、著者が一通り読んだ上で公開している。

本記事は教育・研究目的の情報共有である。投資・法律上の助言ではなく、最新情報は各社公式ドキュメントを直接確認されたい。記事の解釈・運用・コードの利用・その他は読者の側にある。本記事に基づく行動の結果について著者は責任を負わない。EU AI Act（2026/8/2 施行）が定める AI 生成物のラベル表示義務を先取りする形で、冒頭・末尾の双方で AI 利用を明示している。

---

## 1. ブラックスワンと DeFi のギャップ

直近4年で Web3 業界には極稀な大規模喪失事象（ブラックスワン）が頻発している。

| 事象                        | 規模                                    | 発生年     |
| --------------------------- | --------------------------------------- | ---------- |
| Wormhole Bridge ハック      | 約 350 億円流出                          | 2022年2月  |
| Ronin Bridge ハック         | 約 690 億円流出                          | 2022年3月  |
| Terra/Luna エコシステム崩壊 | 約 4.4 兆円喪失（1週間）                  | 2022年5月  |
| Nomad Bridge ハック         | 約 210 億円流出                          | 2022年8月  |
| FTX 取引所破綻              | 顧客資産 約 8,800 億円凍結              | 2022年11月 |

_※ 1ドル = 110円で概算。_

Web3 業界のハッキング被害は単年で数十億ドル規模（Chainalysis: 2024年 約 $2.2B、2025年 約 $3.4B）、本記事冒頭の主要6事例だけで累計 **$400 億超**。これに対し、**DeFi 保険のカバー率は DeFi 全体 TVL の約 0.25%**（市場リーダー Nexus Mutual 基準・Lombard Notes 2026/1 分析）。「保護より利回り・速度を優先し、損失を実験のコストとして受容する」という DeFi 文化が、既存保険の普及を抑制している。

そして、たとえ既存DeFi保険が個別事象をカバーしていても、**保険会社自身の支払い能力が事象規模を超えれば被害者は救済されない**。これが本シリーズの出発点である。

> ※ 公平を期して補足：**ブロックチェーン × 再保険** の先行例は存在する。public chain 側では Uno Re / Lunos の SSRP（2021〜稼働、2026/3 にコンプライアンス基盤へピボット。**公式 GitBook は "manual administration" を明記**しており、人間判断ベース）と Nexus Mutual × Symbiotic 統合（2025/11発表・capital-provision 型）があり、permissioned chain 側では **B3i** が 2022年4月に Allianz × Swiss Re で「DLT 上初の XL再保険契約」を締結している（同年7月コンソーシアム破産・解散）。ただし **public blockchain 上で誰でも参照・改変できる、スマートコントラクトで自動発動する XL（Excess of Loss）型の参照実装** は、2026年5月時点で公開ドキュメントから確認できない。詳細は note 全文版 §3-3 と本シリーズ #2 で整理する。

---

## 2. 伝統保険業界の答え：超過損害額再保険

保険業界は約 **340 年**（Lloyd's of London 1688年〜）かけて、**1社の支払能力を超える巨大災害** への備えを発達させてきた。それが「保険会社のための保険 = 再保険」である。

再保険には大きく2方式ある。

- **比例再保険（Proportional）**：保険料と保険金を一定割合で分け合う
- **超過損害額再保険（Excess of Loss / XL）**：一定額を超えた損害だけを再保険会社が引き受ける

確率は低いが影響甚大 — つまり**釣鐘型の確率分布の端（テールリスク）** に位置する事象に理論的に最適なのは、**超過損害額再保険**である。本シリーズで作るのも、この型の Web3 実装。

再保険業界の世界市場規模は、**グロス保険料収入で年間約 4,080 億米ドル（約 60 兆円・2024年時点）**。決して小さくないインフラ産業である。

---

## 3. Campanella の仕組み

| 要素 | 内容 |
| --- | --- |
| 対象 | 既存DeFi保険会社の **支払い能力を担保する超過損害額再保険** |
| 発動条件 | 既存DeFi保険会社が支払い限界を超える事象が発生したとき |
| 補償フロー | オンチェーンロジックで自動補填 → 被害を受けた個人ユーザーへ最終到達 |
| 速度目標 | **24時間以内に必要な人に届く** |
| 設計思想 | 「機能だけのコード」ではなく **相互扶助構造** の Web3 実装 |

### 概念実装スケッチ（Solidity）

実装の方向感を共有するための最小骨格。**未テスト・概念実装**である。本番ロジックは Stage A 開発時に Foundry テスト・Slither 解析・自己監査を経て確定する。

```solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity ^0.8.24;

import {ERC4626} from "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";

interface ICoverProvider {
    function remainingCapacity() external view returns (uint256);
}

/// Campanella 超過損害額再保険プール（概念実装スケッチ・未テスト）
contract CampanellaReinsurancePool is ERC4626 {
    /// 担保する既存DeFi保険会社
    ICoverProvider public immutable coveredInsurer;
    /// 超過損害発動の閾値（例：USDC建ての一定額）
    uint256 public immutable triggerThreshold;
    /// 被害者ごとの補填残額（24時間以内に claim() で引き出し可能）
    mapping(address victim => uint256 amount) public pendingPayouts;

    event ReinsuranceTriggered(uint256 indexed eventId, uint256 lossAmount);
    event Payout(address indexed victim, uint256 amount);

    /// 既存保険会社の支払い限界超過を検知 → 再保険プール発動
    function triggerReinsurance(uint256 eventId, uint256 lossAmount) external {
        require(coveredInsurer.remainingCapacity() == 0, "insurer still solvent");
        require(lossAmount > triggerThreshold, "below threshold");
        emit ReinsuranceTriggered(eventId, lossAmount);
        // 被害者リストを記録（実装は省略）
    }

    /// 被害者が補填を引き出す
    function claim() external {
        uint256 amount = pendingPayouts[msg.sender];
        require(amount > 0, "no payout pending");
        pendingPayouts[msg.sender] = 0;
        IERC20(asset()).transfer(msg.sender, amount);
        emit Payout(msg.sender, amount);
    }
}
```

ポイント：

- **`ERC4626` 継承** ─ DeFi vault の標準。ステーカーが資金を預けて share トークンを受け取る既存パターンに乗る
- **`ICoverProvider` 抽象化** ─ Nexus Mutual・他プロトコル・将来追加される保険会社すべてに対応可能
- **`triggerThreshold`** ─ 超過損害額再保険の本質。**閾値超過分のみ** が再保険プールから出る
- **`claim()` Pull 方式** ─ 被害者が能動的に引き出す。Push（自動送金）よりガス効率・セキュリティで優れる
- **`pendingPayouts` mapping** ─ 実装では Merkle Tree 化して被害者数千〜数万人にスケールさせる予定

---

## 4. 3段階ロードマップ

| Stage | 時期 | 内容 |
| --- | --- | --- |
| **Stage A**（dApp） | 2026〜2027年 | Solidity + Foundry・Sepolia 限定運用・自己監査公開 |
| **Stage B**（L2 / Appchain） | 2028〜2029年 | OP Stack または Cosmos SDK ・チーム形成・コミュニティ参加 |
| **Stage C**（独自L1） | 2030年〜 | 独自コンセンサス・Apache 2.0 完全オープンソース移行 |

ライセンス戦略は **BSL 1.1 → Apache 2.0 自動移行**（Uniswap・Aave・Sentry が採用している実証済みの3段階モデル）。

---

## 5. 著者について

元大手保険会社で **個人もやる法人営業** を約3年経験。
取得資格：生命保険募集人・損害保険募集人・FP3級・AFP（現在期限切れ）。
現在は **Web3 セキュリティ** に取り組んでおり、Code4rena・Codehawks の参加プロセスを一通り通して **POC を作成済み**。
保険業界と Web3 の交差点で、これから書き手として活動していく。

---

## 6. 次回予告

**#2：今の DeFi 保険が抱えるリスク — 何が守られていて、何が抜け落ちているのか**

既存DeFi保険プロトコル（Nexus Mutual・Unslashed・Etherisc・Sherlock 等）の公開ドキュメントを直接当たり、それぞれが何をカバーし、何をカバーしていないかを整理する。DeFi ユーザー全員にとって、自分の資産を守るための判断材料になる予定。

---

## 全文版・思想・経歴・運営方針

本記事は **要約版** である。
動機の詳細・思想（漫画・音楽との3表現）・著者の詳しい背景・運営方針・ライセンス3段階の解説の全文は、**note 全文版** でお読みいただけます。

📚 **note 全文版**：https://note.com/sleepycat1234/n/n6ecec8b06ade

---

## 関連リンク

- 前作：[【2026年7月終了】Code4rena、Codehawksへの移行ガイド｜Web3スマートコントラクト監査](https://qiita.com/sleepycat_web3/items/03218576b7bf0e4e6b36)
- 📚 note 有料ガイド：[Code4rena 終了。Codehawks に移行する前に知っておくべきこと（¥980）](https://note.com/sleepycat1234/n/n685c954ae14f) — 元保険営業の Codehawks 入門
- 🐦 X：[@lo_cmalu_ractu](https://x.com/lo_cmalu_ractu) — Web3 セキュリティ・本シリーズの進捗を発信
- 🔬 GitHub：Stage A 開始時に公開予定（Business Source License 1.1）

---

## 参考文献

**伝統再保険・歴史**

- Taleb, Nassim Nicholas. _The Black Swan: The Impact of the Highly Improbable_. Random House, 2007.
- Lloyd's of London. _Lloyd's history_. https://www.lloyds.com/about-lloyds/history
- Swiss Re Institute. _sigma 3/2023: World insurance: stirred, and not shaken_. Swiss Re, 2023.

**DeFi 被害統計**

- Chainalysis. _2025 Crypto Crime Mid-Year Update_. https://www.chainalysis.com/blog/2025-crypto-crime-mid-year-update/
- Chainalysis. _2025 Crypto Theft Reaches $3.4 Billion_. https://www.chainalysis.com/blog/crypto-hacking-stolen-funds-2026/

**DeFi 保険・先行プロトコル**

- Lombard Notes. _DeFi's missing primitive: Insurance_ (2026/1). https://lombardnotes.xyz/2026/01/14/defis-missing-primitive-insurance/
- Nexus Mutual Documentation: https://docs.nexusmutual.io
- Nexus Mutual × Symbiotic 統合（2025/11/19）. CoinDesk. https://www.coindesk.com/business/2025/11/19/defi-insurance-alternative-nexus-mutual-integrates-restaking-specialist-symbiotic
- Uno Re. _Single Sided Reinsurance Pool_. https://unore.gitbook.io/uno-re/investor-app/v2-penta-launch-dapp/single-sided-reinsurance-pool
- Lunos. https://lunos.xyz/

**B3i**

- Insurance Journal. _Industry's Blockchain Project, B3i, Ceases to Trade After Filing for Insolvency_ (2022/7/29). https://www.insurancejournal.com/news/international/2022/07/29/677926.htm

---

_本記事は著者個人の見解・経験に基づくものです。記載した数値・事例は2026年5月時点の情報に基づき、最新の状況は各公式情報をご参照ください。先行プロトコル各社の技術仕様（特に SSRP の支払い構造、Symbiotic vault の損失配分ルール）に関する XL／比例の判別は、現時点で公開ドキュメントから完全には確定できないため、続編 #2 で各社ドキュメントを直接当たって再整理する予定です。Nexus Mutual・Unslashed・Etherisc・Sherlock・Uno Re（Lunos）・Symbiotic・B3i・Allianz・Zurich・Aegon・Munich Re・Swiss Re・Hannover Re・Lloyd's of London その他本記事中で言及した各社・各プロトコルとは、資本関係・業務委託・パートナーシップなど一切の関係はありません。各社名・サービス名は各権利者の商標です。本記事の内容は教育・研究目的の情報提供であり、投資助言・法律助言ではありません。_

_思想・アイディア・校閲・校正：cmalu ractu_
_文章生成：Claude（Anthropic）_

---

#Web3 #DeFi #保険 #ブラックスワン #再保険 #スマートコントラクト #ブロックチェーン #保険営業

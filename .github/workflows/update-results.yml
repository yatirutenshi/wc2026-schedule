// =====================================================================
// W杯2026 結果自動更新スクリプト
// football-data.org API から試合結果・決勝Tの対戦カードを取得し、
// matches.json を更新する。GitHub Actions から30分ごとに実行される想定。
//
// 必要な環境変数:
//   FOOTBALL_DATA_TOKEN ... football-data.org の無料APIキー
// =====================================================================
import fs from "node:fs";

const TOKEN = process.env.FOOTBALL_DATA_TOKEN;
const FILE = new URL("../matches.json", import.meta.url).pathname;

// ---- 大会期間外は何もせず終了（Actionsの無駄実行ガード） ----
const now = new Date();
if (now < new Date("2026-06-10T00:00:00Z") || now > new Date("2026-07-21T12:00:00Z")) {
  console.log("大会期間外のためスキップしました。");
  process.exit(0);
}

if (!TOKEN) {
  console.error("環境変数 FOOTBALL_DATA_TOKEN が設定されていません。");
  process.exit(1);
}

// ---- 英語国名 → 日本語表記（football-data.orgのname/shortName両対応） ----
const TEAM_JA = {
  "mexico": "メキシコ", "south africa": "南アフリカ",
  "korea republic": "韓国", "south korea": "韓国", "korea": "韓国",
  "czechia": "チェコ", "czech republic": "チェコ",
  "canada": "カナダ", "bosnia and herzegovina": "ボスニア・ヘルツェゴビナ", "bosnia": "ボスニア・ヘルツェゴビナ",
  "qatar": "カタール", "switzerland": "スイス",
  "brazil": "ブラジル", "morocco": "モロッコ", "haiti": "ハイチ", "scotland": "スコットランド",
  "united states": "アメリカ", "usa": "アメリカ", "united states of america": "アメリカ",
  "paraguay": "パラグアイ", "australia": "オーストラリア",
  "turkey": "トルコ", "turkiye": "トルコ", "türkiye": "トルコ",
  "germany": "ドイツ", "curacao": "キュラソー", "curaçao": "キュラソー",
  "ivory coast": "コートジボワール", "cote d'ivoire": "コートジボワール", "côte d'ivoire": "コートジボワール",
  "ecuador": "エクアドル",
  "netherlands": "オランダ", "japan": "日本", "sweden": "スウェーデン", "tunisia": "チュニジア",
  "belgium": "ベルギー", "egypt": "エジプト", "iran": "イラン", "ir iran": "イラン",
  "new zealand": "ニュージーランド",
  "spain": "スペイン", "cape verde": "カーボベルデ", "cabo verde": "カーボベルデ",
  "saudi arabia": "サウジアラビア", "uruguay": "ウルグアイ",
  "france": "フランス", "senegal": "セネガル", "iraq": "イラク", "norway": "ノルウェー",
  "argentina": "アルゼンチン", "algeria": "アルジェリア", "austria": "オーストリア", "jordan": "ヨルダン",
  "portugal": "ポルトガル",
  "dr congo": "DRコンゴ", "congo dr": "DRコンゴ", "democratic republic of the congo": "DRコンゴ",
  "uzbekistan": "ウズベキスタン", "colombia": "コロンビア",
  "england": "イングランド", "croatia": "クロアチア", "ghana": "ガーナ", "panama": "パナマ"
};

function toJa(name) {
  if (!name) return null;
  const key = name.toLowerCase().normalize("NFKD").replace(/[\u0300-\u036f]/g, "").trim();
  if (TEAM_JA[key]) return TEAM_JA[key];
  // 部分一致フォールバック（"Korea Republic U23" 等の表記ゆれ対策）
  for (const [k, v] of Object.entries(TEAM_JA)) {
    if (key.includes(k)) return v;
  }
  return null;
}

// ---- UTC日時 → JSTの [month, day, "h:mm"] ----
function toJst(utcStr) {
  const t = new Date(Date.parse(utcStr) + 9 * 3600 * 1000);
  return {
    month: t.getUTCMonth() + 1,
    day: t.getUTCDate(),
    time: `${t.getUTCHours()}:${String(t.getUTCMinutes()).padStart(2, "0")}`
  };
}

// ---- スコア文字列の組み立て（PK戦は "1-1 PK4-2" 形式） ----
function buildScore(score) {
  const ft = score?.fullTime;
  if (ft == null || ft.home == null || ft.away == null) return null;
  let s = `${ft.home}-${ft.away}`;
  const pk = score.penalties;
  if (pk && pk.home != null && pk.away != null) s += ` PK${pk.home}-${pk.away}`;
  return s;
}

// ---- メイン ----
const res = await fetch("https://api.football-data.org/v4/competitions/WC/matches", {
  headers: { "X-Auth-Token": TOKEN }
});
if (!res.ok) {
  console.error(`APIエラー: ${res.status} ${res.statusText}`);
  process.exit(1);
}
const api = await res.json();
console.log(`APIから ${api.matches?.length ?? 0} 試合を取得しました。`);

const data = JSON.parse(fs.readFileSync(FILE, "utf8"));
const before = JSON.stringify(data.matches);
let updatedCount = 0, filledCount = 0, unmatched = [];

for (const am of api.matches ?? []) {
  const homeJa = toJa(am.homeTeam?.name) ?? toJa(am.homeTeam?.shortName);
  const awayJa = toJa(am.awayTeam?.name) ?? toJa(am.awayTeam?.shortName);
  const jst = toJst(am.utcDate);

  // --- 行の特定: ①両国名一致（順不同） → ②JSTキックオフ日時一致 ---
  let row = null;
  if (homeJa && awayJa) {
    row = data.matches.find(m =>
      (m[5] === homeJa && m[6] === awayJa) || (m[5] === awayJa && m[6] === homeJa)
    );
  }
  if (!row) {
    const cand = data.matches.filter(m =>
      m[0] === jst.month && m[1] === jst.day && m[2] === jst.time
    );
    // 同時刻キックオフ（GS第3節）は国名でしか特定できないため、候補が1つの時だけ採用
    if (cand.length === 1) row = cand[0];
  }
  if (!row) {
    if (homeJa || awayJa) unmatched.push(`${am.utcDate} ${am.homeTeam?.name} vs ${am.awayTeam?.name}`);
    continue;
  }

  // --- 決勝Tの対戦カード確定: プレースホルダーを国名に置き換え ---
  if (homeJa && awayJa && (row[5] !== homeJa || row[6] !== awayJa)) {
    row[5] = homeJa;
    row[6] = awayJa;
    filledCount++;
  }

  // --- キックオフ時刻の変更にも追従 ---
  if (row[0] !== jst.month || row[1] !== jst.day || row[2] !== jst.time) {
    row[0] = jst.month; row[1] = jst.day; row[2] = jst.time;
  }

  // --- 終了試合のスコア反映 ---
  if (am.status === "FINISHED" || am.status === "AWARDED") {
    const s = buildScore(am.score);
    if (s && row[9] !== s) {
      if (row[8] === undefined) row[8] = null;
      row[9] = s;
      updatedCount++;
    }
  }
}

if (unmatched.length) {
  console.log("⚠ 照合できなかった試合（要確認）:");
  unmatched.forEach(u => console.log("  " + u));
}

if (JSON.stringify(data.matches) === before) {
  console.log("変更なし。コミットはスキップされます。");
  process.exit(0);
}

data.updatedAt = new Date(now.getTime() + 9 * 3600 * 1000).toISOString().replace("Z", "+09:00");
const lines = data.matches.map(m => "    " + JSON.stringify(m));
fs.writeFileSync(FILE, `{\n  "updatedAt": ${JSON.stringify(data.updatedAt)},\n  "matches": [\n${lines.join(",\n")}\n  ]\n}\n`);
console.log(`✅ 更新完了: スコア反映 ${updatedCount} 件 / 対戦カード確定 ${filledCount} 件`);

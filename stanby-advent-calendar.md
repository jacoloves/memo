はじめに

アドベントカレンダー9日目を担当するInfra&Data グループのたなしょです。

普段の業務では、AWSのコスト削減を目的に各リソースの構成を考えたり、EKS周りのモニタリングやAWSリソースの検証を行っています。

最近は趣味でRustやHaskell、Terraformをよく触っています。

なぜ作るのか

AWS関連の調査をする際には、AWS CLIから特定のコマンドを実行してリソースの情報を取得しています。その際、標準出力をパイプラインでつなぎ、jqを使ってJSONの特定箇所を抜き出したり、データをCSV形式で出力する機会が増えてきました。

JSONファイルやCSVファイルを確認するとき、画面を切り替えるのが少し面倒だなと感じることがありました。また、大好きなCUI環境から出たくないという我儘もあり、アドベントカレンダーの記事ネタとしてツールを自作することにしました。

言語は、最近趣味で触っていて楽しいRustを使用することにしました。

作って良かったこと

日頃から使用できるツールができた

Rustで今まで知らなかったライブラリや仕様を理解することができた

日頃から使用できるツールができた

調査を日々実施する中で、素早く使えるツールとして役立っています

CUI環境を離れることなくファイルを確認できるので、ストレスが軽減されました

Rustで今まで知らなかったライブラリや仕様を理解することができた

TUIでよく使うcrosstermライブラリの仕様を少し理解できました（特にenable::terminalが改行される仕様など）

テーブルを形成するためのprettytableライブラリの存在と仕様について学びました

機能ごとにファイルを分割する方法を学びました

最終的に以下の画像ようなCSV/JSONビューワーができました

技術仕様

ファイル構成

プロジェクトは以下のファイルで構成されています。

- Cargo.toml：依存クレートの定義
- src/
  - main.rs：エントリーポイント。ファイルの種類に応じて適切な処理を呼び出します。
  - csv_reader.rs：CSVファイルを読み込み、テーブルデータに変換します。
  - json_reader.rs：JSONファイルを読み込み、テーブルデータに変換します。
  - display.rs：テーブルデータをページング表示し、ユーザー入力を処理します。
  - terminal_utils.rs：ターミナルの設定を制御します（Rawモードの切り替えなど）。
  - utils.rs：汎用的なユーティリティ関数（コマンドライン引数の取得など）を含みます。

機能の概要と実装

1. CSVおよびJSONファイルの読み込み

CSVファイルの読み込み
csvクレートを使用して、CSVファイルを読み込みます。以下は、csv_reader.rsでの実装例です。

wrap_text関数でテキストを折り返し、セルの幅をターミナルに合わせています。

Cellにstyle_spec("FW")を適用して、折り返しを有効にしています。

use csv::ReaderBuilder;
use prettytable::{Cell, Row, Table};
use crossterm::terminal;
use textwrap::wrap;

// テキストを折り返す関数
fn wrap_text(text: &str, max_width: usize) -> String {
    let wrapped_lines = wrap(text, max_width);
    wrapped_lines.join("\n")
}

pub fn read_csv_to_table(file_path: &str) -> Result<(Row, Table), Box<dyn Error>> {
    let file = File::open(file_path)?;
    let mut rdr = ReaderBuilder::new().from_reader(file);

    // ターミナルの幅を取得
    let (term_width, _) = terminal::size()?;
    let max_cell_width = (term_width as usize).saturating_sub(4);

    // ヘッダー行の作成
    let headers = rdr.headers()?;
    let header_cells = headers.iter().map(|h| {
        let wrapped_text = wrap_text(h, max_cell_width);
        let mut cell = Cell::new(&wrapped_text);
        cell.style_spec("FW"); // テキストの折り返しを有効にする
        cell
    }).collect();
    let header_row = Row::new(header_cells);

    // データ行の作成
    let mut table = Table::new();
    for result in rdr.records() {
        let record = result?;
        let cells = record.iter().map(|field| {
            let wrapped_text = wrap_text(field, max_cell_width);
            let mut cell = Cell::new(&wrapped_text);
            cell.style_spec("FW");
            cell
        }).collect();
        table.add_row(Row::new(cells));
    }

    Ok((header_row, table))
}

JSONファイルの読み込み
serde_jsonクレートを使用して、JSONファイルをパースします。以下は、json_reader.rsでの実装例です。

JSONデータが配列であり、その中にオブジェクトが含まれていることを確認しています。

ヘッダーとデータ行のセルに対して、テキストの折り返しを適用しています。

use serde_json::Value;
// 他のインポートは省略

pub fn read_json_to_table(file_path: &str) -> Result<(Row, Table), Box<dyn Error>> {
    // JSONファイルの読み込みとパース
    let file = File::open(file_path)?;
    let json_data: Value = serde_json::from_reader(file)?;

    // ターミナルの幅を取得
    let (term_width, _) = terminal::size()?;
    let max_cell_width = (term_width as usize).saturating_sub(4);

    // JSONが配列であることを確認
    let array = json_data.as_array().ok_or("Expected JSON array at root")?;

    // ヘッダー行の作成
    let headers = array.first()
        .and_then(|item| item.as_object())
        .map(|obj| obj.keys().cloned().collect::<Vec<_>>())
        .ok_or("JSON array does not contain objects")?;

    let header_cells = headers.iter().map(|h| {
        let wrapped_text = wrap_text(h, max_cell_width);
        let mut cell = Cell::new(&wrapped_text);
        cell.style_spec("FW");
        cell
    }).collect();
    let header_row = Row::new(header_cells);

    // データ行の作成
    let mut table = Table::new();
    for item in array {
        if let Some(obj) = item.as_object() {
            let cells = headers.iter().map(|key| {
                let value = obj.get(key).unwrap_or(&Value::Null);
                let cell_text = match value {
                    Value::Null => "".to_string(),
                    Value::String(s) => s.clone(),
                    Value::Number(n) => n.to_string(),
                    Value::Bool(b) => b.to_string(),
                    _ => "".to_string(),
                };
                let wrapped_text = wrap_text(&cell_text, max_cell_width);
                let mut cell = Cell::new(&wrapped_text);
                cell.style_spec("FW");
                cell
            }).collect();
            table.add_row(Row::new(cells));
        }
    }

    Ok((header_row, table))
}

2. テーブル表示の工夫

テーブルのフォーマット設定
prettytableクレートを使用して、テーブルの表示をカスタマイズします。

列の区切りや境界線、パディングを設定することで、見やすいテーブルを実現しています。

use prettytable::format::{FormatBuilder, LinePosition, LineSeparator};

let format = FormatBuilder::new()
    .column_separator('|')
    .borders('|')
    .separator(LinePosition::Top, LineSeparator::new('-', '+', '+', '+'))
    .separator(LinePosition::Title, LineSeparator::new('=', '+', '+', '+'))
    .separator(LinePosition::Bottom, LineSeparator::new('-', '+', '+', '+'))
    .padding(1, 1) // 左右のパディングを設定
    .build();
table.set_format(format);

長い文字列の折り返し
セル内の長い文字列をターミナルの幅に合わせて折り返します。

textwrapクレートのwrap関数を使用しています。

折り返したテキストをセルに設定し、style_spec("FW")で折り返しを有効にしています。

fn wrap_text(text: &str, max_width: usize) -> String {
    let wrapped_lines = wrap(text, max_width);
    wrapped_lines.join("\n")
}

3. ページング機能

display.rsで、ページング表示とユーザー入力の処理を実装しています。

crosstermクレートを使用して、非同期的にキー入力を取得しています。

ユーザーの操作に応じて、表示するページを変更しています。

use crossterm::event::{self, Event, KeyCode};

pub fn display_table(
    header_row: Row,
    table: &Table,
    page_size: usize,
) -> Result<(), Box<dyn Error>> {
    // 省略...

    loop {
        // ページング表示の処理
        // ...

        // キー入力の処理
        if event::poll(std::time::Duration::from_millis(500))? {
            if let Event::Key(key_event) = event::read()? {
                match key_event.code {
                    KeyCode::Down | KeyCode::Char('j') => {
                        // 次のページへ
                    }
                    KeyCode::Up | KeyCode::Char('k') => {
                        // 前のページへ
                    }
                    KeyCode::Char('q') => {
                        // プログラムの終了
                        break;
                    }
                    _ => {}
                }
            }
        }
    }

    Ok(())
}

4. ターミナルサイズへの対応

ターミナルの幅を動的に取得し、それに合わせてセルの折り返し幅を調整しています。

ターミナルのサイズ変更にも対応できるように、表示時に幅を取得しています

// ターミナルの幅を取得
let (term_width, _) = terminal::size()?;
let max_cell_width = (term_width as usize).saturating_sub(4);

5. ターミナル設定の制御

ターミナルをRawモードに切り替え、キー入力を即時に取得できるようにしています。

プログラムの開始時にRawモードを有効化し、終了時に元の設定に戻しています。

use termios::{tcsetattr, Termios, ECHO, ICANON, TCSANOW};
use crossterm::terminal;

pub fn init_terminal() -> Result<Termios, Box<dyn Error>> {
    // 端末設定の取得と変更
}

pub fn reset_terminal(original_termios: &Termios) -> Result<(), Box<dyn Error>> {
    // 端末設定のリセット
}

工夫したポイント

長い文字列の表示  
課題：AWSのARNなど、非常に長い文字列がセルに含まれると、テーブル表示が崩れてしまう。  
解決策：  

テキストの折り返し：textwrapクレートを使用して、セル内のテキストをターミナル幅に合わせて折り返しました。  

セルのスタイル設定：prettytableのセルにstyle_spec("FW")を適用して折り返しを有効にしました。

ユーザー体験の向上  

即時のキー入力反映：ターミナルをRawモードにすることで、ユーザーがキーを押したタイミングで即座にページングが行われるようにしました  

ターミナルサイズの動的対応：ターミナルサイズ変更にも対応できるよう、表示時にサイズを取得するようにしました

エラーハンドリングの強化  

不正なファイル形式への対応：ファイルの拡張子を確認し、サポートしていない形式の場合はエラーメッセージを表示するようにしました。  

JSONの構造チェック：JSONのルートが配列であり、オブジェクトの配列であることを確認しています。

おわりに

日頃行っている行動をCUIツール化できて良かったです。  
Rustを書くことでさまざまなライブラリを理解することができ、非常に勉強になりました。  
今後も業務で使えそうなツールをいろいろと作っていきたいです。  
最後まで読んでいただき、ありがとうございました。

参考

https://docs.rs/prettytable/latest/prettytable/ 

https://docs.rs/crossterm/latest/crossterm/ 

https://docs.rs/csv/latest/csv/ 

https://docs.rs/serde/latest/serde/ 

https://docs.rs/serde_json/latest/serde_json/ 

https://docs.rs/termios/latest/termios/ 

https://docs.rs/textwrap/latest/textwrap/ 

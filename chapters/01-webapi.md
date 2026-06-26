# 第1章：WebAPIの基本

> 執筆者：原 翔悟<br>
> 最終更新：2026-04-22

## この章で学ぶこと

APIからデータを非同期で取得し、アプリ内にリストで表示する方法を学ぶ。
iTunes Search APIを使って音楽を検索して、リスト表示

## 模範コードの全体像

```swift
import SwiftUI

// MARK: - データモデル

struct SearchResponse: Codable {
   let results: [Song]
}

struct Song: Codable, Identifiable {
   let trackId: Int
   let trackName: String
   let artistName: String
   let artworkUrl100: String
   let previewUrl: String?
   
   var id: Int { trackId }
}

// MARK: - メインビュー

struct ContentView: View {
   @State private var songs: [Song] = []
   @State private var searchText: String = ""
   @State private var isLoading: Bool = false
   
   var body: some View {
      NavigationStack {
         VStack {
            // 検索バー
            HStack {
               TextField("アーティスト名を入力", text: $searchText)
                  .textFieldStyle(.roundedBorder)
               
               Button("検索") {
                  Task {
                     await searchMusic()
                  }
               }
               .buttonStyle(.borderedProminent)
               .disabled(searchText.isEmpty)
            }
            .padding(.horizontal)
            
            // 検索結果リスト
            if isLoading {
               ProgressView("検索中...")
                  .padding()
               Spacer()
            } else if songs.isEmpty {
               ContentUnavailableView(
                  "曲を検索してみよう",
                  systemImage: "music.note",
                  description: Text("アーティスト名を入力して検索ボタンを押してください")
               )
            } else {
               List(songs) { song in
                  SongRow(song: song)
               }
            }
         }
         .navigationTitle("Music Search")
      }
   }
   
   // MARK: - API通信
   
   func searchMusic() async {
      guard let encodedText = searchText.addingPercentEncoding(
         withAllowedCharacters: .urlQueryAllowed
      ) else { return }
      
      let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"
      
      guard let url = URL(string: urlString) else { return }
      
      isLoading = true
      
      do {
         let (data, _) = try await URLSession.shared.data(from: url)
         let response = try JSONDecoder().decode(SearchResponse.self, from: data)
         songs = response.results
      } catch {
         print("エラー: \(error.localizedDescription)")
         songs = []
      }
      
      isLoading = false
   }
}

// MARK: - 曲の行ビュー

struct SongRow: View {
   let song: Song
   
   var body: some View {
      HStack(spacing: 12) {
         AsyncImage(url: URL(string: song.artworkUrl100)) { image in
            image
               .resizable()
               .aspectRatio(contentMode: .fill)
         } placeholder: {
            Color.gray.opacity(0.3)
         }
         .frame(width: 60, height: 60)
         .clipShape(RoundedRectangle(cornerRadius: 8))
         
         VStack(alignment: .leading, spacing: 4) {
            Text(song.trackName)
               .font(.headline)
               .lineLimit(1)
            
            Text(song.artistName)
               .font(.subheadline)
               .foregroundStyle(.secondary)
         }
      }
      .padding(.vertical, 4)
   }
}

#Preview {
   ContentView()
}
```

**このアプリは何をするものか：**

iTunes Search APIを使い、音楽（曲）を検索して結果をリスト表示するもの。

## コードの詳細解説

### データモデル（Codable構造体）

```swift
struct SearchResponse: Codable {
   let results: [Song]
}

struct Song: Codable, Identifiable {
   let trackId: Int
   let trackName: String
   let artistName: String
   let artworkUrl100: String
   let previewUrl: String?
   
   var id: Int { trackId }
}
```

**何をしているか：**
JSONからデータを構造体に変換して、保持する際に使うデータ構造を定義している。

**なぜこう書くのか：**
構造体としてJSONの構造を模すと分かりやすく、Swiftの便利なコーダブルがフルに使えるから」

---

### API通信の処理

```swift
func searchMusic() async {
      guard let encodedText = searchText.addingPercentEncoding(
         withAllowedCharacters: .urlQueryAllowed
      ) else { return }
      
      let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"
      
      guard let url = URL(string: urlString) else { return }
      
      isLoading = true
      
      do {
         let (data, _) = try await URLSession.shared.data(from: url)
         let response = try JSONDecoder().decode(SearchResponse.self, from: data)
         songs = response.results
      } catch {
         print("エラー: \(error.localizedDescription)")
         songs = []
      }
      
      isLoading = false
   }
```

**何をしているか：**
検索欄に入力された文字をパーセントエンコーディング（URLで使用できない文字（日本語、スペース、一部の記号）を、「%」と2桁の16進数に変換）している。<br>
guard letで正常に変数を定義できたら、次へ。<br>
URLを文字列で組み立てて、その文字列からURL型に変数を定義。<br>
guard letで正常に変数を定義できたら、次へ。<br>
URLから非同期で通信してデータを取得する。<br>
取得したデータから、構造体「SearchResponse」に当てはめて、JSONとしてデコードする。<br>

**なぜこう書くのか：**
関数として定義するが、非同期としないとViewが固まってしまうため、非同期で書いている。

**もしこう書かなかったら：**
Viewが固まってしまう。


### ビューの構成

```swift

struct ContentView: View {
   @State private var songs: [Song] = []
   @State private var searchText: String = ""
   @State private var isLoading: Bool = false
   
   var body: some View {
      NavigationStack {
         VStack {
            // 検索バー
            HStack {
               TextField("アーティスト名を入力", text: $searchText)
                  .textFieldStyle(.roundedBorder)
               
               Button("検索") {
                  Task {
                     await searchMusic()
                  }
               }
               .buttonStyle(.borderedProminent)
               .disabled(searchText.isEmpty)
            }
            .padding(.horizontal)
            
            // 検索結果リスト
            if isLoading {
               ProgressView("検索中...")
                  .padding()
               Spacer()
            } else if songs.isEmpty {
               ContentUnavailableView(
                  "曲を検索してみよう",
                  systemImage: "music.note",
                  description: Text("アーティスト名を入力して検索ボタンを押してください")
               )
            } else {
               List(songs) { song in
                  SongRow(song: song)
               }
            }
         }
         .navigationTitle("Music Search")
      }
   }
}

struct SongRow: View {
   let song: Song
   
   var body: some View {
      HStack(spacing: 12) {
         AsyncImage(url: URL(string: song.artworkUrl100)) { image in
            image
               .resizable()
               .aspectRatio(contentMode: .fill)
         } placeholder: {
            Color.gray.opacity(0.3)
         }
         .frame(width: 60, height: 60)
         .clipShape(RoundedRectangle(cornerRadius: 8))
         
         VStack(alignment: .leading, spacing: 4) {
            Text(song.trackName)
               .font(.headline)
               .lineLimit(1)
            
            Text(song.artistName)
               .font(.subheadline)
               .foregroundStyle(.secondary)
         }
      }
      .padding(.vertical, 4)
   }
}
```

**何をしているか：**
ContentViewのvar bodyのところで画面全体の構成を書いている。
Viewの構造体である SongRowの var bodyのところで、1曲の表示のレイアウトを設定している。
**なぜこう書くのか：**
Viewを分けて書くことで、管理がしやすくなっている。


（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`Codable` | JSONデータとSwiftの構造体を相互変換するプロトコル、Codable=Encodable+Decodable | `struct Song: Codable { ... }` |
| 例：`async/await` | 非同期処理を同期的に書ける構文 | `let data = try await URLSession.shared.data(from: url)` |
| `Identifiable` | そのデータが、他と区別できる一意のID（識別子）を持っていること | `struct Song: Codable, Identifiable { let trackId: Int}` |
| | | |
| | | |

## 自分の実験メモ

**実験1：**
- やったこと：プロトコルであるCodableの部分をDecodableに変えた
- 結果：今回のプログラムではエラーは出なかった。
- わかったこと：APIを使うときに、なんとなく書いていたCodableの意味が理解できた。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：** コード中の「Identifiable」とは
```Swift
struct Song: Codable, Identifiable {
    let trackId: Int
}
```

   **得られた理解：** 「そのデータが、他と区別できる一意のID（識別子）を持っていること」**を証明するため。リストやForでIdentifiableの構造体を使う時、構造体の中に必ず名前がidのプロパティがなければならない、ということが初めて知った。

3. **質問：** 下記のコードのCodableとはどういう意味？
```Swift
struct SearchResponse: Codable {
let results: [Song]
}
```

   **得られた理解：** 用途はやはりJSON形式とSwiftのデータ構造を相互に変換するため。CodableはEncodableとDecodableの2つを合体させた便利なプロトコル。

## この章のまとめ

ここまで見た目や機能が本格的なものを、この行のコードで書けることに驚いた。
肝はAPI通信のsearchMusic()メソッドとJSONのデコーダブル、URL組み立ての処理である。

# MIYA_DB

<a href="https://github.com/y6-maenaka/miya_core">MiyaCoin</a>向けカスタムKey-Valueデータベースシステム

## 目次

1. [プロジェクト概要](#プロジェクト概要)
2. [アーキテクチャ設計](#アーキテクチャ設計)
3. [技術スタック](#技術スタック)
4. [データ構造の詳細設計](#データ構造の詳細設計)
5. [主要機能とロジック](#主要機能とロジック)
6. [アルゴリズムの詳細](#アルゴリズムの詳細)
7. [ディレクトリ構造とファイル配置](#ディレクトリ構造とファイル配置)
8. [ビルドとセットアップ](#ビルドとセットアップ)
9. [使用例](#使用例)
10. [パフォーマンス特性](#パフォーマンス特性)
---

## プロジェクト概要

**MIYA_DB**は、ブロックチェーンプロジェクト「MiyaCoin」向けに設計された、C++20で実装されたカスタムKey-Valueデータベースシステムです。

### 主要特徴

- **独自のメモリ管理システム**: 5バイトポインタ（optr）で最大8TBのアドレス空間をサポート
- **カスタムB-Tree実装**: SHA-1ハッシュ（20バイト）をキーとした永続化B-Tree
- **セーフモード（トランザクション機構）**: ファイル分離方式による簡易トランザクション、最大5つの同時トランザクションをサポート
- **BSON通信プロトコル**: クエリとレスポンスをBSONフォーマットで送受信
- **ブロックチェーン最適化**: UTXO（未使用トランザクション出力）管理に最適化された設計
- **プラガブルストレージエンジン**: UnifiedStorageManagerインターフェースにより複数のストレージエンジンを切り替え可能

### 設計目的

このデータベースは、以下の用途を想定して設計されています：

1. **MiyaCoinのUTXO管理**: ブロックチェーンの未使用トランザクション出力を効率的に管理
2. **高速なKey-Value操作**: SHA-1ハッシュをキーとした高速な検索・追加・削除
3. **データの永続化**: すべてのデータ構造をファイルに直接配置し、電源断に対応
4. **トランザクションサポート**: セーフモードによる原子性保証

---

## アーキテクチャ設計

### 全体構造

MIYA_DBは**レイヤードアーキテクチャ**と**プラガブルストレージエンジンパターン**を採用しています。

#### 処理フロー

```
Application（外部アプリケーション）
    ↓
ConnectionManager（StreamBufferContainer経由）
    ↓
QueryParser（BSONをQueryContextに変換）
    ↓
Planner（テーブルとストレージエンジンの選定）
    ↓
Executor（ストレージエンジンを用いてデータ操作）
    ├─ IndexManager（B-Tree検索・更新）
    │   └─ OverlayMemoryManager（ファイルI/O）
    │       └─ CacheTable（LRUキャッシュ）
    └─ ValueStoreManager（データ取得・保存）
        └─ OverlayMemoryManager（ファイルI/O）
    ↓
Serializer（結果をBSONレスポンスに変換）
    ↓
ConnectionManager
    ↓
Application
```

参照: `miya_db/flow.txt`

### 主要コンポーネント

#### 1. DatabaseManager（`miya_db/database_manager.cpp/h`）

**役割**: データベースシステムの中核、クエリハンドリング全体を管理

**主要メソッド**:
- `startWithLightMode()`: 軽量モードで起動（スレッド生成）
- クエリタイプ別のディスパッチャー（QUERY_ADD, QUERY_SELECT, QUERY_EXISTS, QUERY_REMOVE）
- SafeModeの管理（QUERY_MIGRATE_SAFE_MODE, QUERY_SAFE_MODE_COMMIT, QUERY_SAFE_MODE_ABORT）

**処理フロー**（`database_manager.cpp:36-244`）:
```cpp
1. StreamBufferContainerからクエリをpop
2. parseQuery(): BSONをQueryContextに変換
3. switch(qctx->type()):
   - QUERY_ADD: mmyisam->add(qctx)
   - QUERY_SELECT: mmyisam->get(qctx)
   - QUERY_EXISTS: mmyisam->exists(qctx)
   - QUERY_REMOVE: mmyisam->remove(qctx)
   - SafeMode関連: migrateToSafeMode/safeCommitExit/safeAbortExit
4. 結果をBSONにシリアライズ
5. StreamBufferContainerにpush
```

#### 2. QueryParser（`miya_db/query_parser/query_parser.cpp/h`）

**役割**: BSONフォーマットのクエリをパース

**主要メソッド**:
- `parseQuery()`: BSONバイナリをQueryContextに変換（`query_parser.cpp:39-132`）
- クエリタイプごとにkey/valueを抽出

**対応クエリタイプ**:
- 1: ADD（データ追加）
- 2: SELECT（データ取得）
- 3: EXISTS（存在確認）
- 4: REMOVE（データ削除）
- 10-12: SafeMode操作（MIGRATE, COMMIT, ABORT）

#### 3. QueryContext（`miya_db/query_context/query_context.h`）

**役割**: クエリの実行コンテキストを保持

**保持データ**:
- `_type`: クエリタイプ
- `_id`: クエリID（uint32_t）
- `_data`: Key/Valueのペア
  - `_key`: キーデータ（SHA-1: 20バイト）
  - `_keyLength`: キー長
  - `_value`: バリューデータ
  - `_valueLength`: バリュー長
- `_safe._index`: RegistryIndex（SafeMode管理用、-1=NormalMode、0-4=SafeMode）
- `_timestamp`: タイムスタンプ

#### 4. MMyISAM（`strage_manager/MMyISAM/MMyISAM.cpp/h`）

**役割**: MySQLのMyISAMにインスパイアされたストレージエンジン実装

**特徴**:
- NormalModeとSafeModeの2つの動作モード
- SafeMode: トランザクション風の操作（最大5同時セッション）
- セーフモードでは別ファイルに書き込み、commit/abortで本体に反映/破棄

**構造**（`MMyISAM.h`）:
```cpp
struct Normal {
    std::shared_ptr<NormalIndexManager> _indexManager;    // B-Treeインデックス管理
    std::shared_ptr<ValueStoreManager> _valueStoreManager; // データ本体管理
}

struct Safe {
    std::array<std::shared_ptr<SafeIndexManager>, 5> _activeSafeIndexManagerArray;     // セーフモード用インデックス
    std::array<std::shared_ptr<ValueStoreManager>, 5> _activeValueStoreManagerArray;   // セーフモード用データストア
}
```

#### 5. UnifiedStorageManager（`strage_manager/unified_storage_manager/unified_storage_manager.h`）

**役割**: ストレージエンジンの抽象インターフェース

**純粋仮想関数**:
- `get()`: データ取得
- `add()`: データ追加
- `remove()`: データ削除
- `exists()`: 存在確認

---

## 技術スタック

| カテゴリ | 技術 |
|---------|------|
| 言語 | C++20 |
| 外部ライブラリ | nlohmann/json（BSON対応） |
| ビルドツール | CMake 3.12以上 |
| 通信プロトコル | BSON（JSON互換バイナリフォーマット） |
| ネットワーク | POSIXソケットAPI |
| ファイルI/O | mmap、標準ファイル操作（open/read/write） |
| 並行処理 | std::thread |
| データ交換 | StreamBuffer（共有メモリバッファ） |

---

## データ構造の詳細設計

### 4.1 オーバーレイポインタ（optr）

**実装ファイル**: `strage_manager/MMyISAM/components/page_table/overlay_ptr.h`

**概要**:
5バイトのカスタムポインタ実装。通常のC++ポインタ（8バイト）より省メモリで、ファイルベースの永続化メモリを指す。

**構造**:
```
[frame (3バイト)] + [offset (2バイト)]
```

- **frame**: ページフレーム番号（上位24ビット）
- **offset**: フレーム内オフセット（下位16ビット）

**最大アドレス空間**:
- 理論値: 2^40 = 1TB
- 実装上の拡張: 約8TB（ページサイズ調整による）

**主要メソッド**:
```cpp
class optr {
    unsigned char _addr[OPTR_ADDR_LENGTH]; // 5バイト
    std::shared_ptr<CacheTable> _cacheTable;

    operator unsigned char*();  // 実際のメモリアドレスに変換
    optr& operator=(const optr& from);
};
```

**特徴**:
- CacheTableへの参照を保持し、アクセス時にページキャッシュを経由
- mmapされたメモリ領域への間接参照を提供
- ファイル永続化に対応した論理アドレス

### 4.2 メモリ管理システム

#### OverlayMemoryManager（`strage_manager/MMyISAM/components/page_table/overlay_memory_manager.h`）

**役割**: ファイルベースのメモリ管理システム

**管理ファイル**:
- データファイル: 実際のデータが格納される
- フリーリストファイル: 空き領域管理情報

**主要メソッド**:
- `allocate(size)`: メモリブロック割り当て → optrを返却
- `deallocate(optr)`: メモリブロック解放
- `dbState()`: データベース状態の取得/設定（SHA-1ハッシュ）
- `syncDBState()`: データベース状態をファイルに同期

**データベース状態（dbState）**:
- 20バイトのSHA-1ハッシュ
- データベースの全体状態を表す
- SafeMode開始時にスナップショット、変更検出に使用（楽観的ロック）

#### OverlayMemoryAllocator（`strage_manager/MMyISAM/components/page_table/overlay_memory_allocator.h`）

**役割**: フリーリスト管理とメモリアロケーション

**メタブロック（MetaBlock）**: 100バイト
```cpp
struct MetaBlock {
    char FORMAT_ID[16];               // "HELLOMIYACOIN!!"
    optr CONTROL_BLOCK_HEAD;          // コントロールブロックチェーンの先頭 (5バイト)
    optr ALLOCATED_BLOCK_HEAD;        // 割り当て済みブロックの先頭 (5バイト)
    optr UNUSED_BLOCK_HEAD;           // 未使用ブロックの先頭 (5バイト)
    unsigned char DB_STATE[20];       // データベース状態 (SHA-1ハッシュ)
    // 残り49バイトは予約領域
};
```

**コントロールブロック（ControlBlock）**: 20バイト
```cpp
struct ControlBlock {
    optr PREV_FREE_BLOCK_OPTR;   // 前のフリーブロック (5バイト)
    optr NEXT_FREE_BLOCK_OPTR;   // 次のフリーブロック (5バイト)
    optr MAPPING_OPTR;           // マッピングポインタ (5バイト)
    optr FREE_BLOCK_END_OPTR;    // フリーブロック終端 (5バイト)
};
```

**割り当てアルゴリズム**: First Fit方式
```cpp
1. フリーブロックチェーンを線形探索
2. 要求サイズ以上の最初のブロックを使用
3. 残りをフリーブロックとして分割
4. コントロールブロックチェーンを更新
```

#### CacheTable（`strage_manager/MMyISAM/components/page_table/cache_manager/cache_table.h`）

**役割**: ページキャッシュ管理

**定数**:
- `CACHE_BLOCK_COUNT`: 3（キャッシュサイズ）

**主要メソッド**:
- `pageIn(frameNumber)`: ディスクからメモリへロード
- `pageOut(frameNumber)`: メモリからディスクへ書き出し
- `pageFault(frameNumber)`: ページフォルトハンドリング
  - LRUアルゴリズムでページを選択
  - ページアウト → ページイン

**LRUアルゴリズム**（`cache_table_LRU.h`）:
- **Clock Algorithm**: LRU近似アルゴリズム
- **参照ビット方式**: 環状リストをポインタで巡回
- **ページアウト**: 参照ビットが0のページを追い出し

### 4.3 B-Treeインデックス構造

#### OBtree（`strage_manager/MMyISAM/components/index_manager/btree.h`）

**役割**: 永続化されたB-Tree実装

**定数定義**:
```cpp
constexpr unsigned int DEFAULT_THRESHOLD = 4;      // 分岐数
constexpr unsigned int KEY_SIZE = 20;              // SHA-1ハッシュ
constexpr unsigned int DATA_OPTR_SIZE = 5;         // データポインタサイズ
constexpr unsigned int NODE_OPTR_SIZE = 5;         // ノードポインタサイズ
```

**主要メソッド**:
- `add(key, dataOptr)`: キーとデータポインタを追加
- `find(key)`: キーを検索 → データポインタとロケーションフラグを返却
- `remove(key)`: キーを削除

#### ONode（`strage_manager/MMyISAM/components/index_manager/o_node.h`）

**役割**: B-Treeノードの表現

**ノード構造**:
```
[親ノードポインタ (5)]
[キー個数 (1)]
[キー配列 (20*3)]
[子個数 (1)]
[子ポインタ配列 (5*4)]
[データポインタ個数 (1)]
[データポインタ配列 (5*3)]
```

**サイズ計算**:
```
O_NODE_ITEMSET_SIZE = 5 + 1 + (20*3) + 1 + (5*4) + 1 + (5*3) = 93バイト
```

**主要操作**:
- `recursiveAdd(key, dataOptr)`: 再帰的な追加とノード分割
- `remove(key)`: キー削除
- `underflow()`: アンダーフロー処理（キー数がTHRESHOLD未満）
- `merge(sibling)`: ノードマージ

**ノード分割アルゴリズム**:
```cpp
1. キー数がTHRESHOLD(4)を超えたら分割
2. 中央のキーを選択
3. 左側のキーと子を左ノードに配置
4. 右側のキーと子を右ノードに配置
5. 中央のキーを親ノードに昇格
6. 親ノードが満杯なら再帰的に分割
```

#### ONodeItemSet（`strage_manager/MMyISAM/components/index_manager/o_node_itemset.h`）

**役割**: ONodeのラッパークラス、読み書き操作を簡素化

**Getter/Setter提供**:
- キー操作: `key(index)`, `keyCount()`, `moveInsertKey()`, `moveDeleteKey()`
- 子ノード操作: `childOptr(index)`, `childOptrCount()`, `moveInsertChildOptr()`
- データポインタ操作: `dataOptr(index)`, `dataOptrCount()`, `moveInsertDataOptr()`

### 4.4 バリューストレージ

#### ValueStoreManager（`strage_manager/MMyISAM/components/value_store_manager/value_store_manager.h`）

**役割**: 実際のデータ値を管理

**特徴**:
- フラグメント方式: データは複数の断片に分割可能（可変長データ対応）
- 各フラグメントは連結リスト構造
- タイムスタンプ付き

**主要メソッド**:
- `add(queryContext)`: データを保存 → データoptrを返却
- `get(dataOptr, &outData)`: データoptrからデータを読み取り
- `mergeDataOptr(conversionTable, safeValueStore)`: SafeモードのデータをNormalにマージ

**ValueFragmentHeader**: 30バイト（デフォルト）
```cpp
struct ValueFragmentHeader {
    uint16_t _headerLength;      // ヘッダー長 (2バイト)
    optr _prevFlagment;          // 前のフラグメント (5バイト)
    optr _nextFlagment;          // 次のフラグメント (5バイト)
    uint64_t _valueLength;       // バリュー長 (8バイト)
    uint64_t _timestamp;         // タイムスタンプ (8バイト)
    // 残り2バイトは予約領域
};
```

**データ格納フロー**:
```cpp
1. ValueFragmentHeaderを作成
2. OverlayMemoryAllocator::allocate()でメモリ割り当て
3. ヘッダーとデータを書き込み
4. データoptrを返却
```

**データ取得フロー**:
```cpp
1. データoptrからValueFragmentHeaderを読み取り
2. _valueLengthを取得
3. フラグメントチェーンを辿る（_nextFlagment）
4. データを結合して返却
```

### 4.5 ユーティリティ関数群

#### optr_utils（`strage_manager/MMyISAM/components/page_table/optr_utils.h`）

**役割**: オーバーレイポインタユーティリティ

**主要関数**:
```cpp
// メモリコピー関数群（optrと通常メモリ間）
optr* omemcpy(optr* dest, optr* src, unsigned long n);
optr* omemcpy(optr* dest, void* src, unsigned long n);
void* omemcpy(void* dest, optr* src, unsigned long n);

// 比較関数
int ocmp(optr* o1, optr* o2, unsigned long size);
int ocmp(optr* o1, void* p2, unsigned long size);
int ocmp(void* p1, optr* o2, unsigned long size);
```

**特徴**: 標準のmemcpy/memcmpのオーバーレイポインタ版として機能

---

## 主要機能とロジック

### 5.1 CRUD操作の詳細

#### ADD操作（`MMyISAM.cpp:49-78`）

**処理フロー**:
```cpp
bool MMyISAM::add(std::shared_ptr<QueryContext> qctx) {
    short registryIndex = qctx->registryIndex();

    if (registryIndex >= 0) {  // SafeMode
        // 1. SafeモードのValueStoreManagerを取得
        auto valueStoreManager = _safe._activeValueStoreManagerArray.at(registryIndex);
        if (valueStoreManager == nullptr) return false;

        // 2. データを保存 → optrを取得
        std::shared_ptr<optr> storeTarget = valueStoreManager->add(qctx);

        // 3. SafeモードのIndexManagerを取得
        auto indexManager = _safe._activeSafeIndexManagerArray.at(registryIndex);
        if (indexManager == nullptr) return false;

        // 4. インデックスにキーとoptrを登録
        indexManager->add(qctx->key(), storeTarget);
    }
    else {  // NormalMode
        // 1. データを保存 → optrを取得
        std::shared_ptr<optr> storeTarget = _normal._valueStoreManager->add(qctx);

        // 2. インデックスにキーとoptrを登録
        _normal._indexManager->add(qctx->key(), storeTarget);

        // 3. データベース状態を更新
        _normal._indexManager->oMemoryManager()->syncDBState();

        // 4. 全SafeModeをクリア（Normal DBが変更されたため）
        clearSafeMode();
    }

    return true;
}
```

**重要なポイント**:
- NormalModeでの追加は、全SafeModeを無効化する（楽観的ロック）
- SafeModeでの追加は、別ファイルに書き込まれるため、Normalには影響しない

#### GET操作（`MMyISAM.cpp:81-117`）

**処理フロー**:
```cpp
bool MMyISAM::get(std::shared_ptr<QueryContext> qctx) {
    short registryIndex = qctx->registryIndex();
    std::pair<std::shared_ptr<optr>, int> dataOptr;
    std::shared_ptr<ValueStoreManager> valueStoreManager;

    // 1. registryIndexに応じてIndexManagerとValueStoreManagerを選択
    if (registryIndex >= 0) {  // SafeMode
        auto safeIndexManager = _safe._activeSafeIndexManagerArray.at(registryIndex);
        if (safeIndexManager == nullptr) return false;

        dataOptr = safeIndexManager->find(qctx->key());
        valueStoreManager = _safe._activeValueStoreManagerArray.at(registryIndex);
    }
    else {  // NormalMode
        dataOptr = _normal._indexManager->find(qctx->key());
        valueStoreManager = _normal._valueStoreManager;
    }

    // 2. データoptrが見つからなければfalseを返却
    if (dataOptr.first == nullptr) return false;

    // 3. データoptrから実データを読み取り
    size_t dataLength;
    std::shared_ptr<unsigned char> data;

    if (dataOptr.second == 0 || dataOptr.second == 1) {
        dataLength = valueStoreManager->get(dataOptr.first, &data);
    }

    // 4. QueryContextに結果をセット
    qctx->value(data, dataLength);
    return true;
}
```

**dataOptr.second（ロケーションフラグ）**:
- 0: NormalModeのデータ
- 1: SafeModeのデータ

#### REMOVE操作（`MMyISAM.cpp:120-146`）

**処理フロー**:
```cpp
bool MMyISAM::remove(std::shared_ptr<QueryContext> qctx) {
    short registryIndex = qctx->registryIndex();

    if (registryIndex >= 0) {  // SafeMode
        auto safeIndexManager = _safe._activeSafeIndexManagerArray.at(registryIndex);
        if (safeIndexManager == nullptr) return false;

        safeIndexManager->remove(qctx->key());
    }
    else {  // NormalMode
        _normal._indexManager->remove(qctx->key());
        clearSafeMode();
    }

    return true;
}
```

**重要な注意点**:
- インデックスからのみキーを削除
- ValueStoreからは削除しない（TODO）
- ディスク領域は解放されない

#### EXISTS操作（`MMyISAM.cpp:149-169`）

**処理フロー**:
```cpp
bool MMyISAM::exists(std::shared_ptr<QueryContext> qctx) {
    std::pair<std::shared_ptr<optr>, int> retDataOptr;
    short registryIndex = qctx->registryIndex();

    if (registryIndex >= 0) {  // SafeMode
        auto indexManager = _safe._activeSafeIndexManagerArray.at(registryIndex);
        if (indexManager == nullptr) return false;

        indexManager->find(qctx->key());
    }
    else {  // NormalMode
        _normal._indexManager->find(qctx->key());
    }

    if (retDataOptr.first == nullptr) return false;
    return true;
}
```

### 5.2 セーフモード（トランザクション機構）

#### 設計思想

**セーフモード**は、MIYA_DBの簡易トランザクション機構です。以下の特徴を持ちます：

1. **ファイル分離方式**: 変更を別ファイルに書き込み
2. **commit時にマージ**: 変更をNormal DBにマージして確定
3. **abort時に破棄**: 変更ファイルを単に破棄
4. **最大5つの同時トランザクション**: registryIndex 0-4
5. **楽観的ロック**: dbStateによる変更検出

**ファイル配置**:
```
table_files/
├── <dbName>/
│   ├── <dbName>           # Normal データファイル
│   └── <dbName>_index     # Normal インデックスファイル
└── .system/safe/<dbName>/
    ├── 0_<dbName>         # registryIndex=0 データファイル
    ├── 0_<dbName>_index   # registryIndex=0 インデックスファイル
    ├── 1_<dbName>         # registryIndex=1 データファイル
    ├── 1_<dbName>_index   # registryIndex=1 インデックスファイル
    ...
```

#### MIGRATE_TO_SAFE_MODE（`MMyISAM.cpp:172-213`）

**目的**: NormalModeからSafeModeへ移行

**処理フロー**:
```cpp
bool MMyISAM::migrateToSafeMode(std::shared_ptr<QueryContext> qctx) {
    short registryIndex = qctx->registryIndex();
    std::string safeDirPath = "../miya_db/table_files/.system/safe/" + _dbName + "/"
                              + std::to_string(registryIndex) + "_";

    // 1. 現在のデータベース状態を保存（スナップショット）
    std::shared_ptr<unsigned char> dbState;
    dbState = _normal._indexManager->oMemoryManager()->dbState();

    // 2. SafeIndexManagerを作成
    //    - NormalBtreeをクローン
    //    - dbStateを保持
    std::string safeIndexFilePath = safeDirPath + _dbName + "_index";
    std::shared_ptr<SafeIndexManager> safeIndexManager =
        std::shared_ptr<SafeIndexManager>(
            new SafeIndexManager(safeIndexFilePath,
                                 _normal._indexManager->masterBtree(),
                                 dbState)
        );
    safeIndexManager->clear();  // 内部でファイルを削除&フォーマット

    // 3. SafeValueStoreManagerを作成
    std::string valueStoreFilePath = safeDirPath + _dbName;
    std::shared_ptr<ValueStoreManager> valueStoreManager =
        std::shared_ptr<ValueStoreManager>(new ValueStoreManager{valueStoreFilePath});
    valueStoreManager->clear();

    // 4. registryIndexに登録
    _safe._activeSafeIndexManagerArray[registryIndex] = safeIndexManager;
    _safe._activeValueStoreManagerArray[registryIndex] = valueStoreManager;

    return true;
}
```

**重要なポイント**:
- `dbState`を保存することで、NormalDBの変更を検出可能に
- `NormalBtree`をクローンすることで、現在のインデックス状態をコピー
- 別ファイルパスで初期化することで、変更を分離

#### SAFE_MODE_COMMIT（`MMyISAM.cpp:216-253`）

**目的**: SafeModeの変更をNormalDBに確定

**処理フロー**:
```cpp
bool MMyISAM::safeCommitExit(std::shared_ptr<QueryContext> qctx) {
    short registryIndex = qctx->registryIndex();

    if (registryIndex < 0) return false;

    // 1. SafeIndexManagerとValueStoreManagerを取得
    std::shared_ptr<SafeIndexManager> safeIndexManager =
        _safe._activeSafeIndexManagerArray.at(registryIndex);
    if (safeIndexManager == nullptr) return false;

    auto valueStoreManager = _safe._activeValueStoreManagerArray.at(registryIndex);
    if (valueStoreManager == nullptr) return false;

    // 2. Meta領域のコピー（100バイト）
    unsigned char addrZero[5]; memset(addrZero, 0x00, sizeof(addrZero));
    std::shared_ptr<ONodeConversionTable> conversionTable = safeIndexManager->conversionTable();
    std::shared_ptr<optr> oNodeMeta =
        std::make_shared<optr>(addrZero, conversionTable->normalOMemoryManager()->dataCacheTable());
    std::shared_ptr<optr> safeONodeMeta =
        std::make_shared<optr>(addrZero, conversionTable->safeOMemoryManager()->dataCacheTable());
    omemcpy(oNodeMeta.get(), safeONodeMeta.get(), 100);

    // 3. データのコピー・移動
    _normal._valueStoreManager->mergeDataOptr(conversionTable, valueStoreManager.get());

    // 4. インデックスのマージ
    std::shared_ptr<ONode> newRootONode =
        dynamic_cast<SafeIndexManager*>(safeIndexManager.get())->mergeSafeBtree();

    // 5. ルートノードの更新
    (_normal._indexManager)->masterBtree()->rootONode(newRootONode);

    // 6. データベース状態を更新
    _normal._indexManager->oMemoryManager()->syncDBState();

    // 7. 全SafeModeをクリア
    clearSafeMode();

    return true;
}
```

**重要なポイント**:
- `ONodeConversionTable`でSafe↔Normalのoptr変換
- `mergeSafeBtree()`で変更されたノードのみをマージ
- `syncDBState()`で新しいデータベース状態を生成

#### SAFE_MODE_ABORT（`MMyISAM.cpp:256-265`）

**目的**: SafeModeの変更を破棄

**処理フロー**:
```cpp
bool MMyISAM::safeAbortExit(std::shared_ptr<QueryContext> qctx) {
    short registryIndex = qctx->registryIndex();
    if (registryIndex < 0) return false;

    // SafeModeをクリア（ファイルは放置）
    clearSafeMode();
    return true;
}
```

**重要なポイント**:
- 単にSafeModeをクリアするだけ
- ファイルは放置される（次回のmigrateで上書き）

#### ONodeConversionTable（`safe_mode_manager/safe_index_manager.h`）

**役割**: Normal↔Safeのoptrマッピングテーブル

**主要メソッド**:
- `entryMap()`: Normal optr → Safe optrのマッピング
- `reverseEntryMap()`: Safe optr → Normal optrのマッピング
- `addEntryMap(normalOptr, safeOptr)`: マッピングを追加

**マージ時の使用例**:
```cpp
1. SafeのB-Treeを走査
2. 各ノードのoptrをONodeConversionTableで変換
3. NormalのB-Treeに反映
```

**コリジョン回避**:
- SafeのノードはNormalとは異なるオフセットに配置
- オフセット: `O_NODE_ITEMSET_SIZE / 2`

### 5.3 クエリAPI（BSONプロトコル）

#### クエリフォーマット

**ADDリクエスト**:
```json
{
    "QueryID": 123,
    "query": 1,
    "key": <binary: 20バイト SHA-1ハッシュ>,
    "value": <binary: 可変長データ>,
    "registryIndex": -1  // -1=NormalMode, 0-4=SafeMode (optional)
}
```

**GETリクエスト**:
```json
{
    "QueryID": 124,
    "query": 2,
    "key": <binary: 20バイト SHA-1ハッシュ>,
    "registryIndex": -1
}
```

**REMOVEリクエスト**:
```json
{
    "QueryID": 125,
    "query": 4,
    "key": <binary: 20バイト SHA-1ハッシュ>,
    "registryIndex": -1
}
```

**EXISTSリクエスト**:
```json
{
    "QueryID": 126,
    "query": 3,
    "key": <binary: 20バイト SHA-1ハッシュ>,
    "registryIndex": -1
}
```

**セーフモード操作**:

MIGRATE（NormalMode → SafeMode）:
```json
{
    "QueryID": 127,
    "query": 10,
    "registryIndex": 0  // 0-4のいずれか
}
```

COMMIT（SafeModeを確定）:
```json
{
    "QueryID": 128,
    "query": 11,
    "registryIndex": 0
}
```

ABORT（SafeModeを破棄）:
```json
{
    "QueryID": 129,
    "query": 12,
    "registryIndex": 0
}
```

#### レスポンスフォーマット

**成功レスポンス**:
```json
{
    "QueryID": 123,
    "status": 1  // 1=成功
}
```

**GETの成功レスポンス**:
```json
{
    "QueryID": 124,
    "status": 1,
    "value": <binary: 取得したデータ>
}
```

**失敗レスポンス**:
```json
{
    "QueryID": 123,
    "status": -1  // -1=失敗
}
```

#### クエリタイプ定義（`miya_db/query_parser/query_type.h`）

```cpp
#define QUERY_ADD 1                   // データ追加
#define QUERY_SELECT 2                // データ取得
#define QUERY_EXISTS 3                // 存在確認
#define QUERY_REMOVE 4                // データ削除
#define QUERY_MIGRATE_SAFE_MODE 10    // SafeMode移行
#define QUERY_SAFE_MODE_COMMIT 11     // SafeModeコミット
#define QUERY_SAFE_MODE_ABORT 12      // SafeModeアボート
```

---

## アルゴリズムの詳細

### 6.1 B-Tree挿入アルゴリズム

**実装ファイル**: `strage_manager/MMyISAM/components/index_manager/btree.cpp`, `o_node.cpp`

**全体フロー**:
```cpp
1. ルートノードから開始
2. リーフノードまで再帰的に探索
3. リーフノードにキーを挿入
4. ノードがオーバーフローしたら分割
5. 分割によるキー昇格を親ノードへ伝播
6. ルートノードが分割されたら新しいルートを作成
```

**詳細アルゴリズム**:

#### ステップ1: リーフノード探索
```cpp
std::shared_ptr<ONode> OBtree::findLeafNode(key) {
    std::shared_ptr<ONode> currentNode = _rootONode;

    while (!currentNode->isLeaf()) {
        // キーより大きい最初の子ノードを選択
        for (int i = 0; i < currentNode->keyCount(); i++) {
            if (ocmp(key, currentNode->key(i), KEY_SIZE) < 0) {
                currentNode = currentNode->childNode(i);
                break;
            }
        }
        // すべてのキーより大きい場合は最後の子ノード
        if (i == currentNode->keyCount()) {
            currentNode = currentNode->childNode(i);
        }
    }

    return currentNode;
}
```

#### ステップ2: キー挿入とノード分割
```cpp
void ONode::recursiveAdd(key, dataOptr) {
    // 適切な位置にキーを挿入
    int insertPos = findInsertPosition(key);
    moveInsertKey(insertPos, key);
    moveInsertDataOptr(insertPos, dataOptr);

    // オーバーフローチェック
    if (keyCount() > DEFAULT_THRESHOLD) {  // THRESHOLD = 4
        split();
    }
}

void ONode::split() {
    // 中央のキーを選択
    int midIndex = keyCount() / 2;
    unsigned char* midKey = key(midIndex);

    // 左ノード作成
    std::shared_ptr<ONode> leftNode = createNode();
    for (int i = 0; i < midIndex; i++) {
        leftNode->moveInsertKey(i, key(i));
        leftNode->moveInsertDataOptr(i, dataOptr(i));
    }

    // 右ノード作成
    std::shared_ptr<ONode> rightNode = createNode();
    for (int i = midIndex + 1; i < keyCount(); i++) {
        rightNode->moveInsertKey(i - midIndex - 1, key(i));
        rightNode->moveInsertDataOptr(i - midIndex - 1, dataOptr(i));
    }

    // 親ノードに中央のキーを昇格
    if (hasParent()) {
        parent()->recursiveAdd(midKey, nullptr);
        parent()->replaceChild(this, leftNode, rightNode);
    } else {
        // ルートノードの分割
        std::shared_ptr<ONode> newRoot = createNode();
        newRoot->moveInsertKey(0, midKey);
        newRoot->setChild(0, leftNode);
        newRoot->setChild(1, rightNode);
        _rootONode = newRoot;
    }
}
```

**計算量**:
- 検索: O(log n)（木の高さ）
- 挿入: O(log n)
- ノード分割: O(THRESHOLD) = O(1)

### 6.2 メモリ割り当てアルゴリズム

**実装ファイル**: `strage_manager/MMyISAM/components/page_table/overlay_memory_allocator.cpp`

**アルゴリズム**: First Fit方式

**全体フロー**:
```cpp
std::shared_ptr<optr> OverlayMemoryAllocator::allocate(size_t size) {
    // 1. フリーブロックチェーンを線形探索
    std::shared_ptr<ControlBlock> currentBlock = _freeBlockHead;

    while (currentBlock != nullptr) {
        size_t blockSize = currentBlock->FREE_BLOCK_END_OPTR - currentBlock->MAPPING_OPTR;

        // 2. 要求サイズ以上の最初のブロックを使用
        if (blockSize >= size) {
            // 3. ブロックを分割
            std::shared_ptr<optr> allocatedOptr = currentBlock->MAPPING_OPTR;

            if (blockSize > size + sizeof(ControlBlock)) {
                // 残りをフリーブロックとして分割
                std::shared_ptr<ControlBlock> newFreeBlock = createControlBlock();
                newFreeBlock->MAPPING_OPTR = allocatedOptr + size;
                newFreeBlock->FREE_BLOCK_END_OPTR = currentBlock->FREE_BLOCK_END_OPTR;

                // フリーリストに追加
                newFreeBlock->NEXT_FREE_BLOCK_OPTR = currentBlock->NEXT_FREE_BLOCK_OPTR;
                currentBlock->NEXT_FREE_BLOCK_OPTR = newFreeBlock;
            }

            // 4. コントロールブロックチェーンを更新
            removeFromFreeList(currentBlock);
            addToAllocatedList(currentBlock);

            return allocatedOptr;
        }

        currentBlock = currentBlock->NEXT_FREE_BLOCK_OPTR;
    }

    // フリーブロックが見つからない → 新しいブロックを割り当て
    return allocateNewBlock(size);
}
```

**割り当て解放（deallocate）**:
```cpp
void OverlayMemoryAllocator::deallocate(std::shared_ptr<optr> ptr) {
    // 1. 割り当て済みリストから削除
    std::shared_ptr<ControlBlock> block = findAllocatedBlock(ptr);
    removeFromAllocatedList(block);

    // 2. フリーリストに追加
    addToFreeList(block);

    // 3. 隣接フリーブロックをマージ
    mergeFreeBlocks(block);
}

void OverlayMemoryAllocator::mergeFreeBlocks(std::shared_ptr<ControlBlock> block) {
    // 前のブロックとマージ
    if (block->PREV_FREE_BLOCK_OPTR != nullptr) {
        std::shared_ptr<ControlBlock> prevBlock = block->PREV_FREE_BLOCK_OPTR;
        if (prevBlock->FREE_BLOCK_END_OPTR == block->MAPPING_OPTR) {
            prevBlock->FREE_BLOCK_END_OPTR = block->FREE_BLOCK_END_OPTR;
            removeFromFreeList(block);
            block = prevBlock;
        }
    }

    // 次のブロックとマージ
    if (block->NEXT_FREE_BLOCK_OPTR != nullptr) {
        std::shared_ptr<ControlBlock> nextBlock = block->NEXT_FREE_BLOCK_OPTR;
        if (block->FREE_BLOCK_END_OPTR == nextBlock->MAPPING_OPTR) {
            block->FREE_BLOCK_END_OPTR = nextBlock->FREE_BLOCK_END_OPTR;
            removeFromFreeList(nextBlock);
        }
    }
}
```

**計算量**:
- 割り当て: O(n)（フリーブロック数に比例）
- 解放: O(1)（隣接ブロックのマージを除く）

### 6.3 LRUキャッシュアルゴリズム

**実装ファイル**: `strage_manager/MMyISAM/components/page_table/cache_manager/cache_table_LRU.cpp`

**アルゴリズム**: Clock Algorithm（LRU近似）

**データ構造**:
```cpp
struct LRU {
    bool _invalidCacheList[CACHE_BLOCK_COUNT];  // 無効フラグ（CACHE_BLOCK_COUNT=3）
    unsigned short _referencePtr = 0;           // 環状リストのポインタ
};
```

**参照時の処理**:
```cpp
void LRU::reference(unsigned short idx) {
    // 参照ビットを立てる
    _invalidCacheList[idx] = false;
}
```

**ページアウト対象の選択**:
```cpp
unsigned short LRU::targetOutPage() {
    while (true) {
        // 1. 現在のポインタが指すページをチェック
        if (_invalidCacheList[_referencePtr]) {
            // 参照ビットが立っている → 次へ
            _invalidCacheList[_referencePtr] = false;
            incrementReferencePtr();
        } else {
            // 参照ビットが立っていない → このページを追い出す
            unsigned short targetIdx = _referencePtr;
            _invalidCacheList[_referencePtr] = true;
            incrementReferencePtr();
            return targetIdx;
        }
    }
}

void LRU::incrementReferencePtr() {
    _referencePtr = (_referencePtr + 1) % CACHE_BLOCK_COUNT;
}
```

**ページフォルト処理**（`cache_table.cpp`）:
```cpp
void CacheTable::pageFault(unsigned long frameNumber) {
    // 1. LRUアルゴリズムで追い出すページを選択
    unsigned short targetIdx = _lru.targetOutPage();

    // 2. ページアウト（ダーティページの場合はディスクに書き出し）
    if (_cacheBlockArray[targetIdx].isDirty()) {
        pageOut(targetIdx);
    }

    // 3. ページイン（ディスクから新しいページを読み込み）
    pageIn(frameNumber, targetIdx);

    // 4. フレームマップを更新
    _frameMap[frameNumber] = targetIdx;
}
```

**計算量**:
- 参照: O(1)
- ページアウト対象選択: O(CACHE_BLOCK_COUNT) = O(1)

### 6.4 セーフモード同期アルゴリズム

**実装ファイル**: `safe_mode_manager/safe_index_manager.cpp`, `safe_btree.cpp`

**主要アルゴリズム**: ONodeConversionTableによるアドレス変換

**dbStateチェック（楽観的ロック）**:
```cpp
bool SafeIndexManager::checkDBState() {
    // SafeMode開始時に保存したdbStateを取得
    std::shared_ptr<unsigned char> savedDBState = _migrationDBState;

    // 現在のNormal DBのdbStateを取得
    std::shared_ptr<unsigned char> currentDBState =
        _conversionTable->normalOMemoryManager()->dbState();

    // 比較
    if (memcmp(savedDBState.get(), currentDBState.get(), 20) != 0) {
        // dbStateが変更されている → Normal DBに変更があった
        return false;  // SafeModeは無効
    }

    return true;  // SafeModeは有効
}
```

**マージ処理**:
```cpp
std::shared_ptr<ONode> SafeIndexManager::mergeSafeBtree() {
    // 1. SafeのB-Treeを走査
    std::shared_ptr<ONode> safeRootNode = _safeBtree->rootONode();

    // 2. 各ノードのoptrをONodeConversionTableで変換
    std::shared_ptr<ONode> normalRootNode = convertSafeToNormal(safeRootNode);

    return normalRootNode;
}

std::shared_ptr<ONode> SafeIndexManager::convertSafeToNormal(
    std::shared_ptr<ONode> safeNode) {

    // 1. SafeノードのoptrをNormalのoptrに変換
    std::shared_ptr<optr> safeOptr = safeNode->selfOptr();
    std::shared_ptr<optr> normalOptr =
        _conversionTable->reverseEntryMap(safeOptr);

    if (normalOptr == nullptr) {
        // 新規ノード → Normalに割り当て
        normalOptr = _conversionTable->normalOMemoryManager()->allocate(O_NODE_ITEMSET_SIZE);
        _conversionTable->addEntryMap(normalOptr, safeOptr);
    }

    // 2. ノードの内容をコピー
    omemcpy(normalOptr.get(), safeOptr.get(), O_NODE_ITEMSET_SIZE);

    // 3. 子ノードを再帰的に変換
    std::shared_ptr<ONode> normalNode =
        std::make_shared<ONode>(normalOptr, _conversionTable->normalOMemoryManager()->dataCacheTable());

    for (int i = 0; i < safeNode->childOptrCount(); i++) {
        std::shared_ptr<ONode> childNode =
            convertSafeToNormal(safeNode->childNode(i));
        normalNode->setChild(i, childNode);
    }

    return normalNode;
}
```

**データのマージ**（`value_store_manager.cpp`）:
```cpp
void ValueStoreManager::mergeDataOptr(
    std::shared_ptr<ONodeConversionTable> conversionTable,
    ValueStoreManager* safeValueStore) {

    // 1. SafeのValueStoreを走査
    // 2. 各データoptrをONodeConversionTableで変換
    // 3. NormalのValueStoreにコピー

    // 実装は複雑なため省略
    // 基本的にはONodeと同様の変換処理
}
```

**計算量**:
- dbStateチェック: O(1)
- マージ処理: O(n)（変更されたノード数に比例）

---

## ディレクトリ構造とファイル配置

### プロジェクト全体構造

```
/Users/yuri/Desktop/miya_db/
├── CMakeLists.txt                          # ビルド設定
├── README.md                               # 本ドキュメント
├── miya_db/                                # アプリケーション層
│   ├── database_manager.cpp/h              # データベースマネージャー（中核）
│   ├── flow.txt                            # アーキテクチャフロー図
│   ├── connection_manager/                 # 接続管理モジュール
│   │   └── connection_manager.h            # TCPソケット通信、MiyaDBMSGプロトコル
│   ├── query_parser/                       # クエリ解析モジュール
│   │   ├── query_parser.cpp/h              # BSONパーサー
│   │   └── query_type.h                    # クエリタイプ定義
│   └── query_context/                      # クエリコンテキスト
│       └── query_context.cpp/h             # クエリデータ保持
├── strage_manager/                         # ストレージエンジン層
│   ├── unified_storage_manager/            # 統一ストレージインターフェース
│   │   └── unified_storage_manager.h       # 抽象基底クラス
│   └── MMyISAM/                           # MyISAMベースのストレージエンジン
│       ├── MMyISAM.cpp/h                   # ストレージエンジン本体
│       ├── components/                     # コンポーネント群
│       │   ├── index_manager/              # インデックス管理
│       │   │   ├── index_manager.h         # 基底クラス
│       │   │   ├── normal_index_manager.h  # 通常モード用
│       │   │   ├── btree.cpp/h             # B-Tree実装
│       │   │   ├── o_node.cpp/h            # ノード実装
│       │   │   └── o_node_itemset.h        # ノードラッパー
│       │   ├── value_store_manager/        # データストレージ
│       │   │   └── value_store_manager.cpp/h
│       │   └── page_table/                 # ページテーブル/メモリ管理
│       │       ├── overlay_memory_manager.cpp/h      # メモリ管理システム
│       │       ├── overlay_memory_allocator.cpp/h    # アロケータ
│       │       ├── overlay_ptr.cpp/h                 # 5バイトポインタ
│       │       ├── optr_utils.cpp/h                  # ユーティリティ
│       │       ├── control_block.cpp/h               # フリーリスト管理
│       │       ├── meta_block.cpp/h                  # メタ情報
│       │       ├── cache_manager/                    # キャッシュ管理
│       │       │   ├── cache_table.cpp/h             # キャッシュテーブル
│       │       │   ├── cache_table_LRU.cpp/h         # LRUアルゴリズム
│       │       │   └── mapper/mapper.cpp/h           # ファイルマッパー
│       │       └── unit_test.cpp/h                   # 単体テスト
│       └── safe_mode_manager/              # セーフモード管理
│           ├── safe_index_manager.cpp/h              # セーフモードインデックス
│           ├── safe_btree.cpp/h                      # セーフモードB-Tree
│           └── safe_o_node.cpp/h                     # セーフモードノード
├── share/                                  # 共有ライブラリ（外部依存）
│   ├── json.hpp                            # nlohmann/json
│   └── stream_buffer/                      # StreamBuffer（共有メモリバッファ）
│       ├── stream_buffer.h
│       └── stream_buffer_container.h
└── table_files/                            # データベースファイル保存領域
    ├── .system/                            # システムファイル
    │   └── safe/                           # セーフモードファイル
    │       └── <dbName>/
    │           ├── 0_<dbName>              # registryIndex=0 データファイル
    │           ├── 0_<dbName>_index        # registryIndex=0 インデックスファイル
    │           ├── 1_<dbName>              # registryIndex=1 データファイル
    │           ├── 1_<dbName>_index        # registryIndex=1 インデックスファイル
    │           └── ...                     # 最大5つまで（0-4）
    ├── block/                              # データベース「block」
    ├── sample/                             # データベース「sample」
    ├── test/                               # データベース「test」
    ├── test_utxo/                          # データベース「test_utxo」
    └── utxo/                               # データベース「utxo」（メイン）
        ├── utxo                            # データファイル
        └── utxo_index                      # インデックスファイル
```

### ファイル詳細説明

#### アプリケーション層

| ファイル | 行数 | 説明 |
|---------|------|------|
| `database_manager.cpp` | 251行 | データベースマネージャー本体、startWithLightMode()でスレッド起動 |
| `database_manager.h` | - | DatabaseManagerクラス定義、QueryParserを継承 |
| `query_parser.cpp` | 145行 | BSONパーサー、parseQuery()でQueryContextに変換 |
| `query_parser.h` | - | QueryParserクラス定義 |
| `query_type.h` | - | クエリタイプ定数定義（QUERY_ADD, QUERY_SELECT等） |
| `query_context.cpp` | - | QueryContextクラス実装 |
| `query_context.h` | - | QueryContextクラス定義 |
| `connection_manager.h` | - | ConnectionManagerクラス定義（実装途中） |
| `flow.txt` | 7行 | アーキテクチャフロー図（テキスト） |

#### ストレージエンジン層

| ファイル | 行数 | 説明 |
|---------|------|------|
| `MMyISAM.cpp` | 292行 | ストレージエンジン本体、CRUD操作、セーフモード管理 |
| `MMyISAM.h` | - | MMyISAMクラス定義 |
| `unified_storage_manager.h` | - | ストレージエンジンインターフェース |

#### インデックス管理

| ファイル | 説明 |
|---------|------|
| `index_manager.h` | インデックス管理の基底クラス |
| `normal_index_manager.h` | 通常モード用インデックス管理 |
| `btree.cpp/h` | B-Tree実装、add/find/remove操作 |
| `o_node.cpp/h` | ノード実装、recursiveAdd/split/merge操作 |
| `o_node_itemset.h` | ノードラッパー、Getter/Setter提供 |

#### ページテーブル/メモリ管理

| ファイル | 説明 |
|---------|------|
| `overlay_memory_manager.cpp/h` | メモリ管理システム、allocate/deallocate/dbState管理 |
| `overlay_memory_allocator.cpp/h` | アロケータ本体、First Fit方式 |
| `overlay_ptr.cpp/h` | 5バイトポインタ実装 |
| `optr_utils.cpp/h` | ユーティリティ関数（omemcpy, ocmp） |
| `control_block.cpp/h` | コントロールブロック、フリーリスト管理 |
| `meta_block.cpp/h` | メタブロック、FORMAT_ID, dbState保持 |
| `cache_table.cpp/h` | キャッシュテーブル、ページイン/アウト |
| `cache_table_LRU.cpp/h` | LRUアルゴリズム実装 |
| `mapper.cpp/h` | ファイルマッパー、mmap操作 |

#### セーフモード管理

| ファイル | 説明 |
|---------|------|
| `safe_index_manager.cpp/h` | セーフモードインデックス管理、ONodeConversionTable保持 |
| `safe_btree.cpp/h` | セーフモードB-Tree、mergeSafeBtree()実装 |
| `safe_o_node.cpp/h` | セーフモードノード |

---

## ビルドとセットアップ

### 前提条件

| 項目 | バージョン |
|------|-----------|
| C++コンパイラ | C++20対応（GCC 10+, Clang 12+） |
| CMake | 3.12以上 |
| nlohmann/json | 最新版 |
| StreamBuffer | カスタムライブラリ（別途用意） |

### ビルド手順

#### 1. 依存関係の配置

```bash
# nlohmann/jsonのダウンロード
cd /Users/yuri/Desktop/miya_db/share/
wget https://github.com/nlohmann/json/releases/download/v3.11.2/json.hpp

# StreamBufferライブラリの配置（別プロジェクト）
cp -r /path/to/stream_buffer /Users/yuri/Desktop/miya_db/share/
```

#### 2. CMakeビルド

```bash
cd /Users/yuri/Desktop/miya_db/
mkdir build && cd build
cmake ..
make
```

#### 3. ビルド成果物

```
build/
└── libMIYA_DB.a    # 静的ライブラリ
```

### CMakeLists.txtの詳細

```cmake
cmake_minimum_required(VERSION 3.12)

add_library(MIYA_DB STATIC
    # ストレージマネージャー
    strage_manager/MMyISAM/MMyISAM.cpp
    strage_manager/unified_storage_manager/unified_storage_manager.cpp

    # インデックス管理
    strage_manager/MMyISAM/components/index_manager/btree.cpp
    strage_manager/MMyISAM/components/index_manager/o_node.cpp
    strage_manager/MMyISAM/components/index_manager/index_manager.cpp
    strage_manager/MMyISAM/components/index_manager/normal_index_manager.cpp

    # ページテーブル
    strage_manager/MMyISAM/components/page_table/overlay_ptr.cpp
    strage_manager/MMyISAM/components/page_table/optr_utils.cpp
    strage_manager/MMyISAM/components/page_table/overlay_memory_manager.cpp
    strage_manager/MMyISAM/components/page_table/overlay_memory_allocator.cpp
    strage_manager/MMyISAM/components/page_table/control_block.cpp
    strage_manager/MMyISAM/components/page_table/meta_block.cpp

    # キャッシュ管理
    strage_manager/MMyISAM/components/page_table/cache_manager/cache_table.cpp
    strage_manager/MMyISAM/components/page_table/cache_manager/cache_table_LRU.cpp
    strage_manager/MMyISAM/components/page_table/cache_manager/mapper/mapper.cpp

    # セーフモード管理
    strage_manager/MMyISAM/safe_mode_manager/safe_index_manager.cpp
    strage_manager/MMyISAM/safe_mode_manager/safe_o_node.cpp
    strage_manager/MMyISAM/safe_mode_manager/safe_btree.cpp

    # バリューストア
    strage_manager/MMyISAM/components/value_store_manager/value_store_manager.cpp

    # アプリケーション層
    miya_db/database_manager.cpp
    miya_db/query_parser/query_parser.cpp
    miya_db/query_context/query_context.cpp

    # テスト
    strage_manager/MMyISAM/components/page_table/unit_test.cpp
    strage_manager/MMyISAM/components/.unit_test.cpp
)

target_include_directories(MIYA_DB PRIVATE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
)

add_definitions(-std=c++20 -w)
```

### ライブラリの使用方法

```cmake
# 外部プロジェクトのCMakeLists.txt
add_subdirectory(/path/to/miya_db)
target_link_libraries(your_project MIYA_DB)
```

---

## 使用例

### 初期化とクエリ送信

```cpp
#include "miya_db/database_manager.h"
#include "share/stream_buffer/stream_buffer_container.h"
#include "share/json.hpp"

int main() {
    // 1. StreamBufferContainerの作成
    auto incomingSBC = std::make_shared<StreamBufferContainer>();
    auto outgoingSBC = std::make_shared<StreamBufferContainer>();

    // 2. DatabaseManagerの初期化
    miya_db::DatabaseManager dbManager;
    dbManager.startWithLightMode(incomingSBC, outgoingSBC, "utxo");

    // 3. クエリ送信スレッドの作成
    std::thread queryThread([&]() {
        // ADDクエリの作成
        nlohmann::json addQuery;
        addQuery["QueryID"] = 1;
        addQuery["query"] = 1;  // QUERY_ADD

        // SHA-1ハッシュのキー（20バイト）
        std::vector<uint8_t> key(20);
        for (int i = 0; i < 20; i++) {
            key[i] = i;
        }
        addQuery["key"] = nlohmann::json::binary(key);

        // バリューデータ
        std::string valueStr = "Hello, MIYA_DB!";
        std::vector<uint8_t> value(valueStr.begin(), valueStr.end());
        addQuery["value"] = nlohmann::json::binary(value);

        addQuery["registryIndex"] = -1;  // NormalMode

        // BSONシリアライズ
        std::vector<uint8_t> bson = nlohmann::json::to_bson(addQuery);

        // StreamBufferに追加
        auto sb = std::make_unique<SBSegment>();
        std::shared_ptr<unsigned char> bsonData(new unsigned char[bson.size()]);
        std::copy(bson.begin(), bson.end(), bsonData.get());
        sb->body(bsonData, bson.size());
        incomingSBC->pushOne(std::move(sb));

        // レスポンス取得
        auto response = outgoingSBC->popOne();
        std::vector<uint8_t> responseBson(
            response->body().get(),
            response->body().get() + response->bodyLength()
        );
        auto responseJson = nlohmann::json::from_bson(responseBson);

        if (responseJson["status"] == 1) {
            std::cout << "ADD成功" << std::endl;
        } else {
            std::cout << "ADD失敗" << std::endl;
        }
    });

    queryThread.join();

    return 0;
}
```

### CRUD操作の完全な例

```cpp
// データ追加（ADD）
nlohmann::json addQuery;
addQuery["QueryID"] = 1;
addQuery["query"] = 1;
addQuery["key"] = nlohmann::json::binary(keyData);
addQuery["value"] = nlohmann::json::binary(valueData);
addQuery["registryIndex"] = -1;

// データ取得（GET）
nlohmann::json getQuery;
getQuery["QueryID"] = 2;
getQuery["query"] = 2;
getQuery["key"] = nlohmann::json::binary(keyData);
getQuery["registryIndex"] = -1;

// 存在確認（EXISTS）
nlohmann::json existsQuery;
existsQuery["QueryID"] = 3;
existsQuery["query"] = 3;
existsQuery["key"] = nlohmann::json::binary(keyData);
existsQuery["registryIndex"] = -1;

// データ削除（REMOVE）
nlohmann::json removeQuery;
removeQuery["QueryID"] = 4;
removeQuery["query"] = 4;
removeQuery["key"] = nlohmann::json::binary(keyData);
removeQuery["registryIndex"] = -1;
```

### セーフモード（トランザクション）の使用例

```cpp
// 1. セーフモードへ移行
nlohmann::json migrateQuery;
migrateQuery["QueryID"] = 10;
migrateQuery["query"] = 10;  // QUERY_MIGRATE_SAFE_MODE
migrateQuery["registryIndex"] = 0;  // 0-4のいずれか
sendQuery(migrateQuery);
auto migrateResponse = receiveResponse();

if (migrateResponse["status"] == 1) {
    // 2. セーフモードで操作
    nlohmann::json addQuery1;
    addQuery1["QueryID"] = 11;
    addQuery1["query"] = 1;  // QUERY_ADD
    addQuery1["key"] = nlohmann::json::binary(key1);
    addQuery1["value"] = nlohmann::json::binary(value1);
    addQuery1["registryIndex"] = 0;  // 同じregistryIndex
    sendQuery(addQuery1);

    nlohmann::json addQuery2;
    addQuery2["QueryID"] = 12;
    addQuery2["query"] = 1;
    addQuery2["key"] = nlohmann::json::binary(key2);
    addQuery2["value"] = nlohmann::json::binary(value2);
    addQuery2["registryIndex"] = 0;
    sendQuery(addQuery2);

    // 3. コミット（確定）
    nlohmann::json commitQuery;
    commitQuery["QueryID"] = 13;
    commitQuery["query"] = 11;  // QUERY_SAFE_MODE_COMMIT
    commitQuery["registryIndex"] = 0;
    sendQuery(commitQuery);
    auto commitResponse = receiveResponse();

    if (commitResponse["status"] == 1) {
        std::cout << "トランザクション確定成功" << std::endl;
    }

    // または、アボート（破棄）
    // nlohmann::json abortQuery;
    // abortQuery["QueryID"] = 14;
    // abortQuery["query"] = 12;  // QUERY_SAFE_MODE_ABORT
    // abortQuery["registryIndex"] = 0;
    // sendQuery(abortQuery);
}
```

---

## パフォーマンス特性

### 計算量

| 操作 | 計算量 | 備考 |
|------|--------|------|
| B-Tree検索 | O(log n) | THRESHOLD=4のため木が深い |
| B-Tree挿入 | O(log n) | ノード分割を含む |
| B-Tree削除 | O(log n) | ノードマージを含む |
| メモリ割り当て | O(m) | mはフリーブロック数（First Fit） |
| メモリ解放 | O(1) | マージを除く |
| LRUページ選択 | O(CACHE_BLOCK_COUNT) = O(1) | Clock Algorithm |
| セーフモードコミット | O(n) | nは変更されたノード数 |

### メモリ使用量

| 項目 | サイズ |
|------|--------|
| optr | 5バイト |
| ONode | 93バイト（固定） |
| MetaBlock | 100バイト |
| ControlBlock | 20バイト |
| ValueFragmentHeader | 30バイト（デフォルト） |
| CacheTable | 3 × ページサイズ |

### ディスクI/O

- **ページキャッシュヒット時**: ディスクI/Oなし
- **ページキャッシュミス時**: 1回のディスク読み込み + （ダーティページの場合）1回のディスク書き込み
- **B-Tree操作**: 最大 log(n) 回のページフォルト

### ボトルネック

1. **キャッシュサイズが小さい（3ブロック）**
   - 頻繁なページフォルトの可能性
   - ディスクI/Oが増加

2. **B-Tree THRESHOLDが小さい（4）**
   - 木が深くなりやすい
   - 検索時のノードアクセス回数が増加

3. **First Fit方式のメモリ割り当て**
   - O(m)の線形探索
   - フリーブロックが多い場合に遅延

4. **セーフモードコミット**
   - 全ノード変換が必要
   - 大量の変更がある場合に時間がかかる



## 参考文献

- MySQL MyISAM ストレージエンジン
- nlohmann/json ライブラリ
- B-Tree アルゴリズム
- LRU キャッシュアルゴリズム

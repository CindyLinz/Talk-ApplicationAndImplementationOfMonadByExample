class: inverse, center, middle

# 實例介紹
# Monad
# 應用設計與實作

## CindyLinz

2016.10.6

---

# 今天的主角是 monad

  + 沒有數學！

  + 不用隱喻！

  + 只有目的, 還有手段!

  + 今天的 monad, 就是 do notation!

---

# 分析會用到的功能

  + 有從屬結構的相依性 - 用空間結構來表達

      - 有一些屬性會依據巢狀結構繼承下來

      - 返回上層時要把內層對屬性的改變丟棄 (或是想成復原)

  + 有時間上的相依性 - 用出現順序來表達

      - 隨著建構過程一步步建立出來的東西, 不能因為「返回上層」而丟棄

      - 筆劃互相覆蓋的順序, 是畫出來的順序, 後面蓋住前面

  + 結構之外的關聯性 - 用名字來指稱

      - 如果需要引用到在之後才會建立的名字, 要先把這一段程式暫停下來先跑別的部分,
        等到指定名字的元件建立以後再跑..

        但是實際畫圖的順序, 仍是依照程式碼排列的順序.

      - 暫停與繼續的部分要考慮時間線被扭曲的現象

---

class: inverse, center, middle

# 準備開始實作

---

## 從狀態核心開始設計

```haskell
data Builder structState agingState a =
  Builder
    ( structState
   -> agingState
   -> (structState, agingState, a)
    )
```

--

  + 定義核心 data type 的形狀, 就決定了這個程式核心的表達能力

      - 它將成為程式開發的軸心, 也劃好能實作的範圍

--

  + Monad type 的形狀, 決定 monad 前端程式中的每一行的能力的最大可能性

      - 這邊說的限制是絕對性的物理極限. 這 type 形狀作不到的事, 就一定作不出來.

      - 這邊有開放允許的能力也不代表前端就有這麼大的能力,
        前端程式也不一定會摸到赤裸裸的 monad 結構,
        我們可以用 module 封裝技巧把它收起來作限制.

---

```haskell
data Builder structState agingState a =
  Builder
    ( structState
   -> agingState
   -> (structState, agingState, a)
    )
instance Functor (Builder structState agingState) where ... (略)
instance Applicative (Builder structState agingState) where ... (略)
instance Monad (Builder structState agingState) where

  return :: a -> Builder structState agingState a

  return a = Builder $ \structState agingState ->
    (structState, agingState, a)


  (>>=)
    :: Builder structState agingState a
    -> (a -> Builder structState agingState b)
    -> Builder structState agingState b

  Builder m >>= f = Builder $ \structState agingState ->
    let
      (structState', agingState', a) =
        m structState agingState
      Builder g = f a
    in
      g structState' agingState'
```

  + 迎合 type 的形狀, 別讓 type 不開心..

---

## 準備一些輔助函數

  + 讓前端不用直接翻弄 monad 結構

```haskell
data Builder structState agingState a =
  Builder
    ( structState
   -> agingState
   -> (structState, agingState, a)
    )

getStruct :: Builder structState agingState structState
getStruct = Builder $ \structState agingState ->
  (structState, agingState, structState)

setStruct :: structState -> Builder structState agingState ()
setStruct newStructState = Builder $ \_ agingState ->
  (newStructState, agingState, ())

modifyStruct
  :: (structState -> structState)
  -> Builder structState agingState ()
modifyStruct f = Builder $ \structState agingState ->
  (f structState, agingState, ())

...
```

---

## 準備一些輔助函數

  + 讓前端不用直接翻弄 monad 結構

```haskell
getStruct :: Builder structState agingState structState
getStruct = Builder $ \structState agingState ->
  (structState, agingState, structState)

setStruct :: structState -> Builder structState agingState ()
setStruct newStructState = Builder $ \_ agingState ->
  (newStructState, agingState, ())

modifyStruct
  :: (structState -> structState)
  -> Builder structState agingState ()
modifyStruct f = Builder $ \structState agingState ->
  (f structState, agingState, ())

...
```

```haskell
do
  struct <- getStruct
  let struct' = ... 對 struct 作一些計算 ...
  setStruct struct'
```

---

## 設定一下結構性狀態與時間性狀態的內容

```haskell
data StructState = StructState
  { sxtTransform :: Matrix
  , sxtFill :: Maybe String
  , sxtStroke :: Maybe String
  , ...
  }
```

```haskell
data AgingState = AgingState
  { tmpDraw :: IO ()
    -- 真正執行作畫動作
  , tmpFill :: Maybe String
    -- 記錄 canvas 2d context 或 cairo 目前指定的顏色
    -- 如果發現 current 顏色與即將要指定的顏色剛好一樣的話
    -- 可以不用重設
  , tmpStroke :: Maybe String
  , ...
  }
```

---

## 準備一些應該會常用的輔助函數

```haskell
setTransform :: Matrix -> Builder StructState agingState ()
getTransform :: Builder StructState agingState Matrix
applyTransform :: Matrix -> Builder StructState agingState ()
applyInvTransform :: Matrix -> Builder StructState agingState ()
...
```

```haskell
identityMatrix :: Matrix
translateMatrix :: Double -> Double -> Matrix
...
```

```haskell
g x y body = do
  let trans = translateMatrix x y
  applyTransform trans
  body
  applyInvTransform trans
  -- 不過反向移動會有些許誤差而回不到原點
```

```haskell
do
  ..畫一畫..
  g 10 10 $ do
    ..偏移 (10,10) 以後再畫一畫
  ..移回來繼續畫一畫..
```

---

## 結構性狀態需要的特殊能力

```haskell
g x y body = local $ do
  applyTransform (translateMatrix x y)
  body
```

  + 回到外層時可以把內層所作的改變丟掉, 恢復進入內層以前的狀態

```haskell
data Builder structState agingState a =
  Builder
    ( structState
   -> agingState
   -> (structState, agingState, a)
    )
```

```haskell
local
  :: Builder structState agingState a
  -> Builder structState agingState a
local (Builder f) = Builder $ \structState agingState ->
  let
    (structState', agingState', a) =
      f structState agingState
  in
    (structState, agingState', a)
    -- 這邊放舊的 structState 而不是新的 structState'
```

---

## 加上結構外關聯用的資訊

```haskell
data Builder info structState agingState a =
  Builder
    ( Maybe String -- 命名中的名字
   -> structState
   -> agingState
   -> M.Map String info -- 取了名字的資訊
   -> (Maybe String, structState, agingState, M.Map String info, a)
    )
```

```haskell
name :: String -> Builder info structState agingState ()
name nextName = Builder $ \ignoredMaybeName structState agingState namedInfo ->
  (Just nextName, structState, agingState, namedInfo, ())

addInfo :: info -> Builder info structState agingState ()
addInfo info = Builder $ \maybeName structState agingState namedInfo ->
  case maybeName of
    Nothing ->
      (maybeName, structState, agingState, namedInfo, ())
    Just name ->
      (Nothing, structState, agingState, M.insert name info namedInfo, ())

queryInfo :: String -> Builder info structState agingState info
queryInfo name = Builder $ \maybeName structState agingState namedInfo ->
  case M.lookup name namedInfo of
    Nothing -> error $ "info of " ++ name ++ " not found."
    Just info -> (maybeName, structState, agingState, namedInfo, info)
```

---

## 設定一下關聯資訊的內容

```haskell
data Pos = Pos Double Double
type Cost = Double
data LinkPoint = LinkPoint Pos Cost

type Info = [LinkPoint]
```

  + 簡單設置為可供連接的連接點候選列表吧

  + Cost 用來設定 preference

  + 這個候選列表讓它無限長, 考慮越多個點結果可以越均勻

--

  + [0,1) 區間的話, 這個序列的分佈不錯:

    ```
    [0, 0.5, 0.25, 0.75, (然後是 奇數/8 們, 奇數/16 們, 奇數/32 們, ...)
    ```

    這個產生器蠻有趣的 (雖然效率不好, θ(N<sup>2</sup>))

    ```haskell
    sepSeries = 0 : inners where
      inners = 0.5 : mix halfs (map (+ 0.5) halfs)
      halfs = map (/ 2) inners
      mix (a:as) (b:bs) = a : b : mix as bs
    ```

---

class: inverse, center, middle

# 準備處理關聯資訊落後定義的問題

```haskell
Nothing -> error $ "info of " ++ name ++ " not found."
```

---

## 保存正在等待的程式

```haskell
data Builder info structState agingState a =
  Builder
    ( Maybe String -- 命名中的名字
   -> structState
   -> agingState
   -> M.Map String info -- 取了名字的資訊
   -> M.Map String [SuspendedBuilder info structState agingState]
   -> ( Maybe String, structState, agingState, M.Map String info
      , M.Map String [SuspendedBuilder info structState agingState]
      , a
      )
    )

data SuspendedBuilder info structState agingState =
  SuspendedBuilder
    ( info -- 需不需要安排這項是 optional 的, 如果無此項, 可以自己重新 query
           -- 有此項可以省一點 query 花的時間, 不過 data type 比較複雜
   -> agingState
   -> M.Map String info
   -> M.Map String [SuspendedBuilder info structState agingState]
   -> ( Maybe String, structState, agingState, M.Map String info
      , M.Map String [SuspendedBuilder info structState agingState]
      , () -- 這邊是 () 不是 a
      )
    )
```

---

## 保存正在等待的程式 (抽出共用部分)

```haskell
data BuilderPart info structState agingState a =
  BuilderPart
    ( agingState
   -> M.Map String info -- 取了名字的資訊
   -> M.Map String [SuspendedBuilder info structState agingState]
   -> ( Maybe String, structState, agingState, M.Map String info
      , M.Map String [SuspendedBuilder info structState agingState]
      , a
      )
    )
data Builder info structState agingState a =
  Builder
    ( Maybe String -- 命名中的名字
   -> structState
   -> BuilderPart info structState agingState a
    )
data SuspendedBuilder info structState agingState =
  SuspendedBuilder
    ( info
   -> BuilderPart info structState agingState ()
    )
```

  + 更突顯 `Builder` 與 `SuspendedBuilder` 的差異處
  + 如果 `SuspendedBuilder` 不用吃 `info` 項的話, `SuspendedBuilder` 就是 `BuilderPart` (但 `a` 處填 `()`)

---

## 讓前端後端合作建構暫停範圍的溝通機制

```haskell
  do
    someInfo <- queryInfo "something"
    ..利用 someInfo 做一些事..

    name "something"
    addInfo [LinkPoint (Pos 0 0) 0]
```

  + 這樣視為使用錯誤

--

```haskell
  do
    fork $ do
      someInfo <- queryInfo "something"
      ..利用 someInfo 做一些事..

    name "something"
    addInfo [LinkPoint (Pos 0 0) 0]
```

  + 正確使用, 以一個 `fork` 包含的範圍劃為暫停的範圍<br>
    (藉用 multi-process / multi-thread 的 fork 函數的名字)

  + `fork` block 的「`a`」不能傳給後面使用, 因為有可能延後執行

     ```
     data SuspendedBuilder info structState agingState -- 沒有 a
     ```

---

## 讓前端後端合作建構暫停範圍的溝通機制

```haskell
(>>=)
  :: Builder info structState agingState a
  -> (a -> Builder info structState agingState b)
  -> Builder info structState agingState b

Builder m >>= f = Builder $ \maybeName structState ->
  BuilderPart $ \agingState namedInfo suspendedBuilder ->
    let
      BuilderPart mPart =
        m maybeName structState
      (maybeName', structState', agingState', namedInfo', suspendedBuilder', a) =
        mPart agingState namedInfo suspendedBuilder
      Builder g =
        f a
      BuilderPart gPart = g maybeName' structState'
    in
      gPart agingState' namedInfo' suspendedBuilder'
```

  + 這是配合剛剛加了那麼多額外欄位作相應變化後的 bind

  + 需要在這裡動點手腳, 它是所謂的 programmable semicolon

---

## 題外話 (儘供參考)

  + 上一頁是比較好理解但是比較繁複的寫法

  + 好寫不好懂的主流(?)寫法如下..

--

```haskell
data BuilderPart = BuilderPart {unBuilderPart :: ...(同前面定義方式)...}
data Builder = Builder {unBuilder :: ...(同前面定義方式)...}

Builder m >>= f = Builder $ \maybeName structState ->
  BuilderPart $ \agingState namedInfo suspendedBuilder ->
    let
      (maybeName', structState', agingState', namedInfo', suspendedBuilder', a)
        = unBuilderPart (m maybeName structState)
          agingState namedInfo suspendedBuilder
    in
      unBuilderPart (unBuilder (f a) maybeName' structState')
        agingState' namedInfo' suspendedBuilder'
```

--

  + 你問說可讀性!?

      - 湊 type 的程式根本就沒有維護的必要..!

      - 我們只讀 type 就飽了.. (各種意味)

---

## 讓前端後端合作建構暫停範圍的溝通機制

```haskell
data SuspendedBuilder info structState agingState a =
  SuspendedBuilder
    ( info
   -> BuilderPart info structState agingState a
    )
```

  + 為了表示建構到一半的 `SuspendedBuilder`, 它的 type 還是需要一個 `a`
  + 建構完畢的才是 `()` 結尾

```haskell
data BuilderPart info structState agingState a
  = BuilderPart
    ( agingState
   -> M.Map String info -- 取了名字的資訊
   -> M.Map String [SuspendedBuilder info structState agingState ()]
   -> Either
      ( String, agingState, M.Map String info
      , M.Map String [SuspendedBuilder info structState agingState ()]
      , SuspendedBuilder info structState agingState a
      ) -- 這是建構暫停範圍的
      ( Maybe String, structState, agingState, M.Map String info
      , M.Map String [SuspendedBuilder info structState agingState ()]
      , a
      ) -- 這是原本的
    )
```

---

## 讓前端後端合作建構暫停範圍的溝通機制

  + 把 `Either`, `Left`, `Right` 換成自己取的名字

```haskell
data BuilderPart info structState agingState a
  = BuilderPart
    ( agingState
   -> M.Map String info -- 取了名字的資訊
   -> M.Map String [SuspendedBuilder info structState agingState ()]
   -> BuilderMode info structState agingState a
    )

data BuilderMode info structState agingState a
  = BuilderExec
    (Maybe String)
    structState
    agingState
    (M.Map String info)
    (M.Map String [SuspendedBuilder info structState agingState ()])
    a
  | BuilderWait
    String -- 正在等待的名字
    agingState -- 暫停以前已經可以畫的部分
    (M.Map String info)
    (M.Map String [SuspendedBuilder info structState agingState ()])
    (SuspendedBuilder info structState agingState a)
```

---

## 讓前端後端合作建構暫停範圍的溝通機制

  + 將 bind 作相應修改 (...深吸一口氣)

--

```haskell
Builder m >>= f = Builder $ \maybeName structState ->
  BuilderPart $ \agingState namedInfo suspendedBuilder ->
    go (mPart agingState namedInfo suspendedBuilder)
    where
      BuilderPart mPart = m maybeName structState
      go ( BuilderExec
           maybeName' structState' agingState'
           namedInfo' suspendedBuilder' a
         ) =
         let
           Builder g = f a
           BuilderPart gPart = g maybeName' structState'
         in
           gPart agingState' namedInfo' suspendedBuilder'
      go ( BuilderWait
           needName agingState'
           namedInfo' suspendedBuilder' (SuspendedBuilder susp)
         ) = BuilderWait needName agingState' namedInfo' suspendedBuilder' $
           SuspendedBuilder $ \info ->
             BuilderPart $ \agingState'' namedInfo'' suspendedBuilder'' ->
               let BuilderPart suspPart = susp info
               in go (suspPart agingState'' namedInfo'' suspendedBuilder'')
```

  + 事後看起來有點可怕, 不過寫的時候感覺還好...<br>
    type 是你路上的光, 跟從 type 的就不在黑暗裡 code..

---

## 讓前端後端合作建構暫停範圍的溝通機制

  + 將 local 作相應修改..

```haskell
local (Builder m) = Builder $ \maybeName structState ->
  BuilderPart $ \agingState namedInfo suspendedBuilder ->
    go (mPart agingState namedInfo suspendedBuilder)
    where
      BuilderPart mPart = m maybeName structState
      go ( BuilderExec
           maybeName' structState' agingState'
           namedInfo' suspendedBuilder' a
         ) =
          BuilderExec maybeName' structState {- 用舊的 structState -}
            agingState' namedInfo' suspendedBuilder' a
      go ( BuilderWait
           needName agingState'
           namedInfo' suspendedBuilder' (SuspendedBuilder susp)
         ) = BuilderWait needName agingState' namedInfo' suspendedBuilder' $
           SuspendedBuilder $ \info ->
             BuilderPart $ \agingState'' namedInfo'' suspendedBuilder'' ->
               let BuilderPart suspPart = susp info
               in go (suspPart agingState'' namedInfo'' suspendedBuilder'')
```

  + 主要的複雜結構和 bind 一樣, 都是遇到 `BuilderWait` 的時候要鑽進最深處的 `BulderExec`

---

## 讓前端後端合作建構暫停範圍的溝通機制

```haskell
mapBuilderPart
  :: ( Maybe String -> structState -> agingState -> M.Map String info
    -> M.Map String [SuspendedBuilder info structState agingState ()]
    -> a -- 此函數的參數部分就是 BuilderExec 裡會拿到的各項
    -> BuilderMode info structState agingState b
     )
  -> BuilderPart info structState agingState a
  -> BuilderPart info structState agingState b
```

  + 把 `local` 與 bind 裡面共同的複雜結構拿出來,<br>
    type 看起來就是對 `BuilderPart` 作 `map`

---

## 讓前端後端合作建構暫停範圍的溝通機制

```haskell
mapBuilderPart f (BuilderPart mPart) =
  BuilderPart $ \agingState namedInfo suspendedBuilder ->
    go (mPart agingState namedInfo suspendedBuilder)
    where

      go ( BuilderExec
           maybeName structState agingState
           namedInfo suspendedBuilder a
         ) = f maybeName structState agingState namedInfo suspendedBuilder a

      go ( BuilderWait
           needName agingState
           namedInfo suspendedBuilder (SuspendedBuilder susp)
         ) = BuilderWait needName agingState namedInfo suspendedBuilder $
           SuspendedBuilder $ \info ->
             BuilderPart $ \agingState' namedInfo' suspendedBuilder' ->
               let BuilderPart suspPart = susp info
               in go (suspPart agingState' namedInfo' suspendedBuilder')

forBuilderPart = flip mapBuilderPart
```

  + `forBuilderPart` 是變換參數順序的版本, f 通常蠻長的, 放在後面比較方便

---

## 讓前端後端合作建構暫停範圍的溝通機制

  + 然後 bind 與 `local` 就好看了

```haskell
Builder m >>= f = Builder $ \maybeName structState ->
  forBuilderPart (m maybeName structState) $
    \maybeName' structState' agingState' namedInfo' suspendedBuilder' a ->
      let
        Builder g = f a
        BuilderPart gPart = g maybeName' structState'
      in
        gPart agingState' namedInfo' suspendedBuilder'

local (Builder m) = Builder $ \maybeName structState ->
  forBuilderPart (m maybeName structState) $
    \maybeName' structState' agingState' namedInfo' suspendedBuilder' a ->
      BuilderExec
        maybeName' structState {- 用舊的 structState -}
        agingState' namedInfo' suspendedBuilder' a
```

  + 不看不好看的地方, 就會覺得變好看..

---

## 讓前端後端合作建構暫停範圍的溝通機制

  + 接下來是 `queryInfo` 裡找不到的情況, 告訴 monad 自己要開始暫停了<br>
    (之前放 error 的地方)

```haskell
queryInfo :: String -> Builder info structState agingState info
queryInfo name =
  Builder $ \maybeName structState ->
    BuilderPart $ \agingState namedInfo suspendedBuilder->

      case M.lookup name namedInfo of
        Just info ->
          BuilderExec maybeName structState
            agingState namedInfo suspendedBuilder info

        Nothing ->
          BuilderWait name agingState namedInfo suspendedBuilder $
            SuspendedBuilder $ \info ->
              BuilderPart $ \agingState' namedInfo' suspendedBuilder' ->
                BuilderExec maybeName structState
                  agingState' namedInfo' suspendedBuilder' info
```

---

## 讓前端後端合作建構暫停範圍的溝通機制

  + 接下來是 `fork`

```haskell
fork
  :: Builder info structState agingState ()
    -- 或是 Builder info structState agingState a 也可以
    -- 不過放在那裡的東西無論是什麼, 註定是要作廢的 [1]
  -> Builder info structState agingState ()
fork (Builder m) =
  Builder $ \maybeName structState ->
    BuilderPart $ \agingState namedInfo suspendedBuilder ->
      let
        BuilderPart mPart = m maybeName structState
      in
        case mPart agingState namedInfo suspendedBuilder of
          BuilderWait needName agingState' namedInfo' suspendedBuilder' susp ->
            BuilderExec
              Nothing structState agingState' namedInfo'
              (M.insertWith (++) needName [susp] suspendedBuilder') ()
              -- 如果 [1] 處是 a 而不是 () 的話,
              -- 這邊要加一個把 susp 的 a 丟棄換成 () 的動作
          BuilderExec maybeName' structState'
              agingState' namedInfo' suspendedBuilder' _ ->
            BuilderExec
              Nothing structState agingState' namedInfo'
              suspendedBuilder' ()
```

---

## 讓前端後端合作建構暫停範圍的溝通機制

  + `addInfo` 時如果發現新加入的 info 正好有程式(們)在等, 將它們喚醒

```haskell
addInfo :: info -> Builder info structState agingState ()
addInfo info =
  Builder $ \maybeName structState ->
    BuilderPart $ \agingState namedInfo suspendedBuilder ->
      case maybeName of
        Nothing -> BuilderExec maybeName structState
          agingState namedInfo suspendedBuilder ()
        Just name -> ... -- 有加到 info 的時候
```

---

## 讓前端後端合作建構暫停範圍的溝通機制

  + `addInfo` 時如果發現新加入的 info 正好有程式(們)在等, 將它們喚醒

```haskell
Just name ->
  BuilderExec Nothing structState agingState' namedInfo' suspendedBuilder' ()
    where
    (agingState', namedInfo', suspendedBuilder') =
      case M.lookup name suspendedBuilder of
        Nothing -> (agingState, M.insert name info namedInfo, suspendedBuilder)
        Just susps -> ... -- 有人在等這個 info 的時候
```

---

## 讓前端後端合作建構暫停範圍的溝通機制

  + `addInfo` 時如果發現新加入的 info 正好有程式(們)在等, 將它們喚醒

```haskell
Just susps -> go agingState
  (M.insert name info namedInfo)
  (M.delete name suspendedBuilder) susps
  where
    go agingState namedInfo suspendedBuilder susps =
      case susps of
        [] -> (agingState, namedInfo, suspendedBuilder)
        SuspendedBuilder susp : susps ->
          let BuilderPart sPart = susp info
          in case sPart agingState namedInfo suspendedBuilder of

            -- 拿了這個 info 就滿足了
            BuilderExec maybeName' structState' agingState'
                namedInfo' suspendedBuilder' _ ->
              go agingState' namedInfo' suspendedBuilder' susps

            -- 拿了這個 info 以後又遇到別的缺乏的 info
            BuilderWait needName agingState'
                namedInfo' suspendedBuilder' susp' ->
              go agingState' namedInfo'
                (M.insertWith (++) needName [susp'] suspendedBuilder') susps
```

---

## 讓 agingState 改變的順序與程式碼的順序一致

  + 雖然 `queryInfo` 會造成寫在前面的程式可能暫停, 但我們希望它作畫的動作還是先於寫在它之後的程式的動作

  + 順序是由後端處理的, 所以將接 `agingState` 變化的順序改由後端負責, 前端只負責它的內容部分

  + `agingState` 加上 `Monoid agingState` 限制, 後端將用 `(<>)` 來接畫圖片段..

--

```haskell
data AgingBuilder agingState = AgingBuilder
  { agingBuilderIter :: Integer -- 目前位置要「下筆」的地方
  , agingBuilderSegs :: (L.LinkedList Integer agingState) -- 間斷的畫圖動作
  }

data BuilderPart info structState agingState a = BuilderPart
  { unBuilderPart ::
    ( AgingBuilder agingState
   -> M.Map String info -- 取了名字的資訊
   -> M.Map String [SuspendedBuilder info structState agingState ()]
   -> BuilderMode info structState agingState a
    )
  }
```

---

## 讓 agingState 改變的順序與程式碼的順序一致

```haskell
data BuilderMode info structState agingState a
  = BuilderExec
    (Maybe String)
    structState
    (AgingBuilder agingState)
    (M.Map String info)
    (M.Map String [SuspendedBuilder info structState agingState ()])
    a
  | BuilderWait
    String -- 正在等待的名字
    (AgingBuilder agingState) -- 留好暫停部位的洞, 以及指定可以先繼續畫的部位
    (M.Map String info)
    (M.Map String [SuspendedBuilder info structState agingState ()])
    (SuspendedBuilder info structState agingState a)
```

---

## 讓 agingState 改變的順序與程式碼的順序一致

`queryInfo` 的修改
```diff
  Nothing ->
-   BuilderWait name agingState namedInfo suspendedBuilder $
+   BuilderWait name agingStateNewHole namedInfo suspendedBuilder $
      SuspendedBuilder $ \info ->
-       BuilderPart $ \agingState' namedInfo' suspendedBuilder' ->
+       BuilderPart $ \(AgingBuilder _ agingSegs') namedInfo' suspendedBuilder' ->
          BuilderExec maybeName structState
-           agingState' namedInfo' suspendedBuilder' info
+           (AgingBuilder iter agingSegs') namedInfo' suspendedBuilder' info
+   where
+     AgingBuilder iter agingSegs = agingState
+     agingSegsNewHole = L.insertAfter iter mempty agingSegs
+     agingStateNewHole =
+       AgingBuilder (L.next agingSegsNewHole iter) agingSegsNewHole
```

---

## 讓 agingState 改變的順序與程式碼的順序一致

`addInfo` 的修改
```diff
  (agingState'', namedInfo'', suspendedBuilder'') =
    case M.lookup name suspendedBuilder of
      Nothing -> (agingState, namedInfo', suspendedBuilder)
      Just susps -> go agingState namedInfo'
        (M.delete name suspendedBuilder) susps
        where
+         AgingBuilder iter _ = agingState
          go agingState namedInfo suspendedBuilder susps =
            case susps of
-             [] -> (agingState, namedInfo, suspendedBuilder)
+             [] -> (AgingBuilder iter agingSegs, namedInfo, suspendedBuilder)
+               where
+                 AgingBuilder _ agingSegs = agingState
              SuspendedBuilder susp : susps ->
                let BuilderPart sPart = susp info
                in case sPart agingState namedInfo suspendedBuilder of
```

---

## 建構 agingState 的程式碼讀取舊的 agingState

  + 原本的 `getAgingState` 去掉了, 因為現在有一堆不完整的 `agingState` 片段,
    無法給出一個已經接完整了的 `agingState`

--

  + 前端可以把 `agingState` 作成這樣的形狀:
    ```haskell
    type AgingState agingState = agingState -> agingState

    instance Monoid (AgingState agingState) where
      mempty = id
      mappend = (.)
    ```

  + 這樣最後拿出來的就是一個 `agingState -> agingState` 的大函數,
    把 init 狀態的 `agingState` 塞進去就可以拿到 final 的 `agingState`..

    此時的 `agingState` 通常會有部分欄位是狀態, 然後其中一個欄位是畫圖動作的 `IO ()`,
    把那個 `IO ()` 拿出來跑就會畫出畫面了.

---

## 結語

  + monad 是 Haskell 非常重要的東西, 幾乎就是 Haskell 特色的一大半,
    最常見的 logo 版本就是一個 lambda + `>>=`
    ![Haskell LOGO](haskell-logo.svg)

--

  + 理解 monad 有許多角度

    由工程的角度來看 monad 就是 do-notation, 就是 programmable semicolon, 依據我們後端的實作來決定前端程式的語意

--

  + 自訂語言來解決問題的方法, 用在目標規格靈活多變的情況, 讓開規格的人自己來說他現在要什麼,
    然後把他講的話自動變成成品.

    而我們盡量創造一個讓他說起來方便一點的空間..

--

  + 我們是工具的主人, 我們決定工具應該如何被使用;<br>
    而不是讓工具限制我們能怎麼用..

    唯一能限制我們的, 是我們有限的想像力.

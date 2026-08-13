# fub_3


檢索演算法


你的認知大方向是對的，但要改一個關鍵說法：不是「大型資料才有更厲害的檢索演算法，小型資料用這些演算法沒有效」，而是 大型通用服務通常有能力建立更完整的 multi-stage retrieval stack，而且某些方法在資料規模變大後，投入的成本才更有價值。

我查了 OpenAI 官方資料與 IR 論文後，我會建議你把這件事拆成三層來理解。

1. ChatGPT / 大型服務，確實通常不只是 Hybrid Search

先講有公開證據的部分。OpenAI 並沒有公開 ChatGPT Search 完整的底層 ranking 演算法，所以不能直接說「ChatGPT Search 一定用了 BM25 + Vector + ColBERT + 某某演算法」。

但 OpenAI 官方已經透露兩件重要事情。

ChatGPT Search 會把使用者問題改寫成一個或多個 targeted queries，看第一次搜尋結果後，還可能再產生更精確的搜尋。這已經不是單純「query → vector search → Top-K」了。

更直接的是 OpenAI 的 hosted File Search。官方文件明確寫出它會：

Query Rewrite
→ Complex Query Decomposition / parallel searches
→ Keyword Search + Semantic Search
→ Reranking
→ LLM Generation


User Query
   ↓
Query Understanding
   ↓
Query Rewrite
   ↓
Complex Query Decomposition
   ↓
┌──────────────────────────┐
│ Keyword / Lexical Search │
│ Semantic / Vector Search │
└──────────────────────────┘
   ↓
Candidate Pool
   ↓
Reranking
   ↓
Top Relevant Chunks
   ↓
LLM

所以，如果你們現在的架構大概是：

Query
 ↓
斷詞 / Entity Extraction
 ↓
BM25 ───┐
        ├→ RRF → Top-N
Vector ─┘
 ↓
LLM

那麼跟大型 hosted retrieval service 比，真正的差距很可能不在「Hybrid Search 這四個字」，而是在 Hybrid Search 前後到底還有多少層策略。


問題

「大型通用 AI 搜尋服務因面對 Open-domain、未知資料來源與複雜 research query，需要較完整的 planning、multi-query、iterative search 與 reflection；企業知識檢索則是 bounded-domain retrieval。在受控且高品質的企業 corpus 中，研究顯示 Hybrid Retrieval、Reranking、Metadata/Context 與 Query Rewrite 即可取得非常具競爭力的結果；只有在 multi-hop、證據不足或跨來源推理等情境下，再啟動 Agentic Retrieval。因此自建 Agentic RAG 的合理方向不是複製大型通用搜尋架構，而是建立『可選擇性升級』的 Adaptive Retrieval 架構。」



大型通用 AI 搜尋服務因面對 Open-domain、未知資料來源與複雜 research query，需要較完整的 planning、multi-query、iterative search 與 reflection；企業知識檢索則是 bounded-domain retrieval。在受控且高品質的企業 corpus 中，研究顯示 Hybrid Retrieval、Reranking、Metadata/Context 與 Query Rewrite 即可取得非常具競爭力的結果；只有在 multi-hop、證據不足或跨來源推理等情境下，再啟動 Agentic Retrieval。因此自建 Agentic RAG 的合理方向不是複製大型通用搜尋架構，而是建立『可選擇性升級』的 Adaptive Retrieval 架構。」

研究結果正在支持一個很重要的觀點：RAG 並不是 Agentic 越多、Search Loop 越多、HyDE / Multi-query 越多就一定越好。對受控、特定領域 corpus，設計良好的 Hybrid Retrieval + Reranker，甚至可能比更複雜的 Agentic / Adaptive Retrieval 更好。

1. 最適合你拿來當證據：金融資料上，Hybrid + Rerank 反而打贏很多複雜方法

2026 年的 From BM25 to Corrective RAG: Benchmarking Retrieval Strategies for Text-and-Table Documents，直接在金融 QA benchmark 上比較 10 種 retrieval strategy，包含 BM25、Dense、Hybrid、HyDE、Multi-Query、Contextual Retrieval、CRAG（Corrective RAG）與 Reranking。資料規模是 23,088 個 queries、7,318 個文件


Hybrid + Reranker > CRAG > Multi-query > HyDE



而且差距不是很小。Hybrid + Reranker 的 Recall@5 達到 0.816，MRR@3 為 0.605；論文作者指出它大幅超過所有 single-stage 方法



如果你要跟副總解釋，我會把核心訊息定義成：大型通用 AI 搜尋服務因面對 Open-domain、未知資料來源與複雜 research query，需要較完整的 planning、multi-query、iterative search 與 reflection；企業知識檢索則是 bounded-domain retrieval。在受控且高品質的企業 corpus 中，研究顯示 Hybrid Retrieval、Reranking、Metadata/Context 與 Query Rewrite 即可取得非常具競爭力的結果；只有在 multi-hop、證據不足或跨來源推理等情境下，再啟動 Agentic Retrieval。因此自建 Agentic RAG 的合理方向不是複製大型通用搜尋架構，而是建立『可選擇性升級』的 Adaptive Retrieval 架構。」



                  User Question
                        ↓
               Semantic Understanding
                        ↓
              Query Strategy Planning
                        ↓
        ┌───────────────┼───────────────┐
        ↓               ↓               ↓
 Query Rewrite    Query Expansion   Decomposition
        ↓               ↓               ↓
      Query 1         Query 2        Sub-Q1/Q2
        │               │               │
        └───────────────┼───────────────┘
                        ↓
                Multi-Retriever
       ┌────────────────┼─────────────────┐
       ↓                ↓                 ↓
     BM25          Dense Vector      Learned Sparse
       │                │                 │
       └────────────────┼─────────────────┘
                        ↓
                  Candidate Pool
                        ↓
                Fusion / Filtering
                        ↓
                Strong Reranker
                        ↓
              Evidence Validation
                        ↓
           足夠？ ─ No → 再搜尋
             │
            Yes
             ↓
          Generation
參考大型通用 AI 的先進檢索技術，但不直接複製其複雜架構；在企業知識檢索則是受控下依最新技術研究下，採Adaptive方式，持續優化Agentic RAG能力模組。


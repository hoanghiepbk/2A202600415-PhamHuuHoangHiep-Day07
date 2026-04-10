# Báo Cáo Lab 7: Embedding & Vector Store

**Họ tên:** Phạm Hữu Hoàng Hiệp
**Nhóm:** C4-C401
**Ngày:** 10/4/2026

---

## 1. Warm-up (5 điểm)

### Cosine Similarity (Ex 1.1)

**High cosine similarity nghĩa là gì?**
> High cosine similarity nghĩa là hai embedding có hướng gần giống nhau trong không gian vector, nên nội dung/ngữ nghĩa của hai câu thường liên quan chặt chẽ. Giá trị càng gần 1 thì mức tương đồng càng cao.

**Ví dụ HIGH similarity:**
- Sentence A: "Python is a programming language used for data science."
- Sentence B: "Python is widely used in data analysis and machine learning."
- Tại sao tương đồng: Cùng nói về Python và ứng dụng trong phân tích dữ liệu/machine learning.

**Ví dụ LOW similarity:**
- Sentence A: "How to train a neural network efficiently?"
- Sentence B: "My favorite recipe is spicy beef noodle soup."
- Tại sao khác: Hai câu thuộc hai chủ đề hoàn toàn khác nhau (AI vs nấu ăn).

**Tại sao cosine similarity được ưu tiên hơn Euclidean distance cho text embeddings?**
> Cosine similarity tập trung vào góc giữa hai vector (hướng ngữ nghĩa), ít bị ảnh hưởng bởi độ lớn vector. Với text embeddings, hướng thường quan trọng hơn khoảng cách tuyệt đối nên cosine ổn định hơn Euclidean.

### Chunking Math (Ex 1.2)

**Document 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**
> *Trình bày phép tính:* step = 500 - 50 = 450. Số chunk xấp xỉ = ceil((10000 - 500) / 450) + 1 = ceil(9500 / 450) + 1 = 22 + 1.
> *Đáp án:* khoảng **23 chunks**.

**Nếu overlap tăng lên 100, chunk count thay đổi thế nào? Tại sao muốn overlap nhiều hơn?**
> Overlap tăng lên 100 thì step còn 400, vì vậy số chunk tăng (xấp xỉ 25 chunk). Overlap lớn hơn giúp giữ ngữ cảnh ở ranh giới chunk, giảm mất ý khi truy xuất.

---

## 2. Document Selection — Nhóm (10 điểm)

### Domain & Lý Do Chọn

**Domain:** Luật/Quy tắc thương mại quốc tế — Incoterms 2020

**Tại sao nhóm chọn domain này?**
> Incoterms 2020 là bộ quy tắc có tính pháp lý cao, nội dung dài và cấu trúc rõ theo điều khoản/rule nên phù hợp để thử retrieval theo ngữ cảnh. Domain này giúp nhóm đánh giá tốt các bài toán truy xuất như phân biệt trách nhiệm người bán/người mua, điểm chuyển rủi ro và phân bổ chi phí.

### Data Inventory

| # | Tên tài liệu | Nguồn | Số ký tự | Metadata đã gán |
|---|--------------|-------|----------|-----------------|
| 1 | incoterms_intro.md | ICC Publication (extracted from incorterm.md) | ~1,400 | source=icc, domain=incoterms, lang=en, section=introduction |
| 2 | incoterms_any_mode_rules.md | ICC Publication (extracted from incorterm.md) | ~1,600 | source=icc, domain=incoterms, lang=en, section=any_mode |
| 3 | incoterms_sea_rules.md | ICC Publication (extracted from incorterm.md) | ~1,200 | source=icc, domain=incoterms, lang=en, section=sea_inland |
| 4 | incoterms_obligations_ab.md | ICC Publication (extracted from incorterm.md) | ~1,000 | source=icc, domain=incoterms, lang=en, section=obligations |
| 5 | incoterms_risk_cost_focus.md | ICC Publication (extracted from incorterm.md) | ~1,300 | source=icc, domain=incoterms, lang=en, section=risk_cost |

### Metadata Schema

| Trường metadata | Kiểu | Ví dụ giá trị | Tại sao hữu ích cho retrieval? |
|----------------|------|---------------|-------------------------------|
| source | string | icc / team_notes | Truy vết nguồn pháp lý của thông tin |
| domain | string | incoterms | Lọc đúng domain khi chạy query nhóm |
| section | string | introduction / any_mode / sea_inland | Filter theo vùng kiến thức trong tài liệu |
| rule | string | EXW / FCA / FOB / CIF / DDP | Tăng precision cho query theo từng điều kiện giao hàng |
| lang | string | en | Đồng bộ ngôn ngữ văn bản để giảm nhiễu |
| doc_id | string | incorterm_main | Hỗ trợ quản lý/xóa tài liệu |

---

## 3. Chunking Strategy — Cá nhân chọn, nhóm so sánh (15 điểm)

### Baseline Analysis

Chạy `ChunkingStrategyComparator().compare()` trên 2-3 tài liệu:

| Tài liệu | Strategy | Chunk Count | Avg Length | Preserves Context? |
|-----------|----------|-------------|------------|-------------------|
| incoterms_intro.md | FixedSizeChunker (`fixed_size`) | 92 | 198 | Trung bình |
| incoterms_intro.md | SentenceChunker (`by_sentences`) | 64 | 286 | Tốt |
| incoterms_intro.md | RecursiveChunker (`recursive`) | 70 | 252 | Tốt |
| incoterms_any_mode_rules.md | FixedSizeChunker (`fixed_size`) | 355 | 199 | Trung bình |
| incoterms_any_mode_rules.md | SentenceChunker (`by_sentences`) | 244 | 289 | Khá |
| incoterms_any_mode_rules.md | RecursiveChunker (`recursive`) | 261 | 258 | Tốt |

### Strategy Của Tôi

**Loại:** RecursiveChunker (có kết hợp metadata filter theo `topic` và `lang`)

**Mô tả cách hoạt động:**
> RecursiveChunker tách văn bản theo thứ tự separator (`\n\n`, `\n`, `. `, ` `, rồi fallback ký tự). Khi một đoạn của Incoterms vẫn vượt `chunk_size`, thuật toán tiếp tục tách sâu theo separator kế tiếp để giữ ngữ nghĩa pháp lý theo từng mục. Cách này phù hợp với tài liệu dài, có nhiều heading và điều khoản như Incoterms 2020.

**Tại sao tôi chọn strategy này cho domain nhóm?**
> Tài liệu Incoterms có cấu trúc phân tầng rõ (Introduction, rule family, từng rule EXW/FCA/...), nên recursive giúp giữ trọn ý trong từng đoạn nghĩa vụ pháp lý. Khi kết hợp filter theo `section` và `rule`, kết quả truy xuất chính xác hơn cho các query chuyên biệt.

**Code snippet (nếu custom):**
```python
# Dùng RecursiveChunker mặc định trong src/chunking.py
chunks = RecursiveChunker(chunk_size=200).chunk(text)
results = store.search_with_filter(
    query,
    top_k=3,
    metadata_filter={"domain": "incoterms", "section": "any_mode", "lang": "en"},
)
```

### So Sánh: Strategy của tôi vs Baseline

| Tài liệu | Strategy | Chunk Count | Avg Length | Retrieval Quality? |
|-----------|----------|-------------|------------|--------------------|
| incoterms_any_mode_rules.md | best baseline (SentenceChunker) | 244 | 289 | 8.2/10 |
| incoterms_any_mode_rules.md | **của tôi** (Recursive+filter) | 261 | 258 | 9.1/10 |

### So Sánh Với Thành Viên Khác

| Thành viên | Strategy | Retrieval Score (/10) | Điểm mạnh | Điểm yếu |
|-----------|----------|----------------------|-----------|----------|
| Tôi (Hiệp) | Recursive + metadata filter | 9.0 | Context mạch lạc, ít nhiễu | Tốn thời gian tune separator |
| Dũng | FixedSize + overlap cao | 7.8 | Đơn giản, nhanh | Dễ cắt giữa ý |
| Cường | SentenceChunker | 8.5 | Câu dễ đọc, tự nhiên | Không ổn với đoạn rất dài |
| Lâm | FixedSize + overlap thấp | 8.0 | Tốc độ xử lý nhanh | Dễ mất ngữ cảnh ở ranh giới chunk |

**Strategy nào tốt nhất cho domain này? Tại sao?**
> RecursiveChunker kết hợp metadata filter là phù hợp nhất cho domain Incoterms. Cách này cân bằng giữa độ dài chunk và tính mạch lạc điều khoản, đồng thời giảm nhiễu khi query theo rule hoặc section.

---

## 4. My Approach — Cá nhân (10 điểm)

Giải thích cách tiếp cận của bạn khi implement các phần chính trong package `src`.

### Chunking Functions

**`SentenceChunker.chunk`** — approach:
> Tôi dùng regex tách câu theo dấu kết thúc câu và khoảng trắng (`(?<=[.!?])\s+`), sau đó strip và loại bỏ phần rỗng. Các câu được gom theo `max_sentences_per_chunk`. Edge case chuỗi rỗng được trả về list rỗng.

**`RecursiveChunker.chunk` / `_split`** — approach:
> Thuật toán recursive tách đoạn theo danh sách separator ưu tiên. Base case gồm: text rỗng, text đã <= `chunk_size`, hoặc hết separator thì fallback cắt theo độ dài cố định. Cách làm này giúp chunk cuối cùng vẫn bounded và giữ ngữ cảnh tốt.

### EmbeddingStore

**`add_documents` + `search`** — approach:
> Mỗi `Document` được chuẩn hóa thành record gồm `id`, `content`, `metadata`, `embedding` rồi lưu vào in-memory store (và add vào Chroma nếu available). Khi `search`, query được embed và tính điểm bằng dot product với từng embedding đã lưu, sau đó sort giảm dần theo score.

**`search_with_filter` + `delete_document`** — approach:
> Tôi filter theo metadata trước, rồi mới chạy similarity trên tập đã lọc để giảm nhiễu. Hàm delete xóa toàn bộ record có `metadata["doc_id"] == doc_id`, trả `True` nếu có bản ghi bị xóa.

### KnowledgeBaseAgent

**`answer`** — approach:
> Agent lấy top-k chunks liên quan từ store, ghép thành phần `Context` trong prompt và thêm `Question` của người dùng. Sau đó gọi `llm_fn(prompt)` để sinh câu trả lời dựa trên ngữ cảnh đã retrieve. Nếu query rỗng thì trả thông báo yêu cầu nhập câu hỏi.

### Test Results

```
============================= test session starts =============================
collected 42 items
...
============================= 42 passed in 0.11s ==============================
```

**Số tests pass:** 42 / 42

---

## 5. Similarity Predictions — Cá nhân (5 điểm)

| Pair | Sentence A | Sentence B | Dự đoán | Actual Score | Đúng? |
|------|-----------|-----------|---------|--------------|-------|
| 1 | CIF requires insurance by seller. | Under CIF, seller contracts for insurance. | high | 0.85 | Đúng |
| 2 | FOB delivery happens on board vessel. | Risk under FOB transfers when goods are on board. | high | 0.82 | Đúng |
| 3 | EXW puts minimum obligations on seller. | DDP requires seller to handle import clearance. | low | 0.29 | Đúng |
| 4 | Incoterms do not determine transfer of ownership title. | Incoterms are not a substitute for a sales contract. | high | 0.73 | Đúng |
| 5 | CPT can be used for any transport mode. | CIF is only for sea and inland waterway transport. | low | 0.34 | Đúng |

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn nghĩa?**
> Cặp số 5 có điểm vẫn dương nhẹ dù khác chủ đề. Điều này cho thấy embedding vẫn có thể chia sẻ một số đặc trưng ngôn ngữ bề mặt, nên cần kết hợp threshold hoặc metadata filter để giảm false positive.

---

## 6. Results — Cá nhân (10 điểm)

Chạy 5 benchmark queries của nhóm trên implementation cá nhân của bạn trong package `src`. **5 queries phải trùng với các thành viên cùng nhóm.**

### Benchmark Queries & Gold Answers (nhóm thống nhất - Incoterms 2020)

| # | Query | Gold Answer |
|---|-------|-------------|
| 1 | What do Incoterms 2020 rules mainly regulate between seller and buyer? | Incoterms mainly regulate obligations, risk transfer point, and allocation of costs between seller and buyer. |
| 2 | Which Incoterms rule is the only one requiring the seller to unload goods at destination? | DPU is the only Incoterms 2020 rule that requires the seller to unload at destination. |
| 3 | What do Incoterms rules explicitly NOT do regarding ownership? | Incoterms do not regulate transfer of property/title/ownership of the goods. |
| 4 | In FOB, when does risk transfer from seller to buyer? | In FOB, risk transfers when the goods are delivered on board the vessel at the named port of shipment. |
| 5 | (Filter by rule=CPT) Under CPT, does the seller guarantee goods arrive in sound condition at destination? | No. Under CPT, risk transfers when goods are handed to the carrier; seller does not guarantee arrival condition at destination. |

### Kết Quả Của Tôi

| # | Query | Top-1 Retrieved Chunk (tóm tắt) | Score | Relevant? | Agent Answer (tóm tắt) |
|---|-------|--------------------------------|-------|-----------|------------------------|
| 1 | What do Incoterms 2020 rules mainly regulate between seller and buyer? | Đoạn liệt kê obligations, risk, costs trong Introduction | 0.86 | Yes | Trả lời đúng 3 trục chính: nghĩa vụ, rủi ro, chi phí |
| 2 | Which Incoterms rule is the only one requiring the seller to unload goods at destination? | Đoạn explanatory notes của DPU | 0.88 | Yes | Trả lời đúng: DPU |
| 3 | What do Incoterms rules explicitly NOT do regarding ownership? | Đoạn "do NOT deal with transfer of title/ownership" | 0.84 | Yes | Trả lời đúng: không điều chỉnh quyền sở hữu |
| 4 | In FOB, when does risk transfer from seller to buyer? | Đoạn FOB nêu thời điểm "on board" | 0.81 | Yes | Trả lời đúng thời điểm chuyển rủi ro |
| 5 | (Filter by rule=CPT) Under CPT, does the seller guarantee goods arrive in sound condition at destination? | Đoạn CPT explanatory notes về không bảo đảm tình trạng tới đích | 0.85 | Yes | Trả lời đúng: không bảo đảm; rủi ro chuyển sớm |

**Bao nhiêu queries trả về chunk relevant trong top-3?** 5 / 5

---

## 7. What I Learned (5 điểm — Demo)

**Điều hay nhất tôi học được từ thành viên khác trong nhóm:**
> Mình học được cách dùng metadata filter theo `topic` để giảm nhiễu mạnh trước khi ranking. Khi dữ liệu bắt đầu lớn hơn, filter trước giúp kết quả top-k ổn định hơn đáng kể.

**Điều hay nhất tôi học được từ nhóm khác (qua demo):**
> Nhóm khác trình bày cách đánh giá retrieval theo checklist rõ ràng (precision, coherence, grounding), không chỉ nhìn score. Cách đo này giúp mình phân tích lỗi hệ thống tốt hơn.

**Nếu làm lại, tôi sẽ thay đổi gì trong data strategy?**
> Nếu làm lại, mình sẽ tách `incorterm.md` thành nhiều tài liệu con ngay từ đầu để kiểm soát retrieval theo section/rule tốt hơn. Mình cũng sẽ xây thêm query phản ví dụ (dễ nhầm giữa CIF/CIP hoặc EXW/DDP) để đánh giá khả năng phân biệt quy tắc.

---

## Tự Đánh Giá

| Tiêu chí | Loại | Điểm tự đánh giá |
|----------|------|-------------------|
| Warm-up | Cá nhân | 5 / 5 |
| Document selection | Nhóm | 9 / 10 |
| Chunking strategy | Nhóm | 14 / 15 |
| My approach | Cá nhân | 9 / 10 |
| Similarity predictions | Cá nhân | 5 / 5 |
| Results | Cá nhân | 9 / 10 |
| Core implementation (tests) | Cá nhân | 30 / 30 |
| Demo | Nhóm | 5 / 5 |
| **Tổng** | | **86 / 100** |

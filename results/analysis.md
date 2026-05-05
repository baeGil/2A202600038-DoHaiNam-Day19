# Báo Cáo Phân Tích Hệ Thống GraphRAG - Lab Day 19
Đỗ Hải Nam - 2A202600038
## 1. Tổng quan hệ thống
Hệ thống được xây dựng để so sánh hiệu quả giữa **Flat RAG** (truy xuất vector truyền thống) và **GraphRAG** (truy xuất kết hợp đồ thị tri thức).
- **Vector DB**: Milvus (Local).
- **Graph DB**: Neo4j (Local).
- **LLM**: gpt-5.4-mini dùng cho trích xuất thực thể và trả lời câu hỏi.
- **Embedding**: jina-embeddings-v5-text-small dùng cho vectorDB.
- **Dữ liệu**: Dataset [Wiki100k](https://huggingface.co/datasets/chonkie-ai/wikipedia-100k/viewer/default/train?p=1), nhưng chỉ lấy 1 sample là row đầu tiên: tổng 38 chunks.

## 2. Hình ảnh đồ thị tri thức
![Knowledge Graph Visualization](figures/neo4j/detailed.png)

## 3. Bảng so sánh kết quả Benchmark (20 câu hỏi)

| Tiêu chí | Flat RAG | GraphRAG |
| :--- | :---: | :---: |
| Số câu hỏi chính xác | 17/20 | 19/20 |
| Độ chính xác (%) | 85% | 95% |
| Số câu GraphRAG thắng tuyệt đối | - | 3 |
| Số câu Flat RAG thắng tuyệt đối | 1 | - |
| Số câu hòa (Cả hai cùng đúng/sai) | 16 | 16 |

**Nhận xét:** GraphRAG cho kết quả vượt trội ở các câu hỏi yêu cầu kết nối thông tin đa bước (multi-hop) và hiểu ngữ cảnh sâu rộng hơn, trong khi Flat RAG đôi khi bị giới hạn bởi phạm vi của một chunk đơn lẻ.

## 4. Phân tích chi phí và hiệu năng xây dựng đồ thị (Indexing Cost)

Dưới đây là phân tích chi phí tài nguyên và thời gian để xây dựng đồ thị tri thức từ dữ liệu đầu vào:

### 4.1. Thống kê dữ liệu đầu vào
- **Số lượng tài liệu:** 1 file markdown (Wiki sample).
- **Số lượng Chunks:** 38 chunks.
- **Kích thước chunk:** 1400 ký tự (overlap 180 ký tự).

### 4.2. Ước tính Token Usage
Quá trình trích xuất thực thể và quan hệ (Entity & Relationship Extraction) tiêu tốn nhiều token nhất:
- **Input Tokens:** 38 chunks × ~1200 tokens/prompt ≈ **45,600 tokens**.
- **Output Tokens:** 38 chunks × ~450-600 tokens/kết quả trích xuất ≈ **17,100 - 22,800 tokens**.
- **Tổng cộng:** Khoảng **62,700 - 68,400 tokens**.
- **Thực tế:** Quá trình thực nghiệm mất khoảng **71k tokens** cho 1 sample này.
- **Chi phí (gpt-5.4-mini):** Với giá $0.75/1M input và $4.5/1M output, tổng chi phí hết **$0.1368 USD** cho 1 sample này (Khá đắt nếu số lượng sample tăng lên).

### 4.3. Phân tích thời gian (Build Time)
- **Embedding Chunks:** < 4 giây (Sử dụng Jina API với batching).
- **LLM Extraction:** Do thực hiện tuần tự, mỗi chunk mất khoảng 4 - 8 giây để LLM xử lý và trả về cấu trúc JSON.
- **Neo4j Upsert:** < 0.1 giây cho mỗi lần cập nhật node/relationship.
- **Tổng thời gian xây dựng (38 chunks):** Khoảng **2 - 4 phút**.

**Thực nghiệm:**
- **Tổng thời gian xây dựng:** khoảng **5 phút**
- **Đánh giá:** Lâu hơn ước tính một chút, khá lâu nếu số lượng chunk tăng lên.

### 4.4. Đánh giá khả năng mở rộng
- Với tập dữ liệu nhỏ (38 chunks), chi phí và thời gian là không đáng kể. 
- Tuy nhiên, khi mở rộng lên 100k samples, thời gian xây dựng có thể lên tới hàng chục ngày nếu không sử dụng xử lý song song (async/parallel processing) và chi phí LLM sẽ trở thành một yếu tố cần cân nhắc kỹ (ước tính ~$13.7k cho toàn bộ tập Wiki100k, phải tối ưu bằng cách sử dụng model rẻ hơn)

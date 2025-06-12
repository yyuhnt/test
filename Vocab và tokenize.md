## Tạo Vocab
### Nguồn
#### Normal vocab

- Lấy vocab từ deratives của http header: [https://github.com/pichillilorenzo/known-http-header-db](https://github.com/pichillilorenzo/known-http-header-db)
    - Content-type của web: Trích từ Sec-List: [https://github.com/danielmiessler/SecLists/blob/master/Discovery/Web-Content/web-all-content-types.txt](https://github.com/danielmiessler/SecLists/blob/master/Discovery/Web-Content/web-all-content-types.txt)
    - Các chuỗi trong User-Agent: [https://github.com/danielmiessler/SecLists/tree/master/Fuzzing/User-Agents](https://github.com/danielmiessler/SecLists/tree/master/Fuzzing/User-Agents)
    - Content-Language: [https://www.localeplanet.com/api/codelist.json](https://www.localeplanet.com/api/codelist.json)

#### Abnormal vocab

- Lấy từ vocab.json của codebert
- Wordlist path traversal của PayloadAllTheThing
    - [](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Directory%20Traversal/Intruder)[https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Directory](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Directory) Traversal/Intruder
- Lấy các keyword trong sql
    - [https://pypi.org/project/sqlparse/](https://pypi.org/project/sqlparse/)
- Dataset CISO


### Nội dung viết
#### 3.3.1 Mục tiêu và vai trò của tập từ vựng

Tập từ vựng (vocabulary) là thành phần then chốt trong quá trình mã hoá chuỗi đầu vào phục vụ huấn luyện mô hình học sâu. Việc ánh xạ mỗi token về một chỉ số duy nhất không chỉ giúc chuẩn hoá dữ liệu, mà còn hỗ trợ mô hình học được các đặc trưng ngữ nghĩa một cách có hệ thống. Bên cạnh đó, xây dựng một tập từ vựng phù hợp còn đóng vai trò quan trọng trong việc giảm nhiễu, giới hạn không gian tìm kiếm và nâng cao hiệu quả huấn luyện. Đặc biệt, trong bối cảnh bài toán phát hiện tấn công thời gian thực trên hệ thống dịch vụ, từ vựng cần đảm bảo phản ánh đầy đủ cả hành vi hợp lệ và các mẫu tấn công tiềm ẩn.

#### 3.3.2 Nguồn thu thập từ vựng

Bước đầu tiên trong quá trình xây dựng từ vựng là trích xuất trực tiếp các token từ tập dữ liệu ATRDF (Cisco, 2024). Việc này đảm bảo rằng toàn bộ từ vựng đều có xuất hiện thực tế trong dữ liệu huấn luyện và phản ánh đúng ngữ cảnh của hệ thống dịch vụ mà mô hình cần bảo vệ. Tuy nhiên, do tập dữ liệu này bị giới hạn về phạm vi và không bao quát đầy đủ các biểu thức có thể gặp phải trong môi trường vận hành thực tế, việc mở rộng từ vựng từ các nguồn bên ngoài là cần thiết.

Để tăng tính tổng quát và khả năng thích ứng với các tình huống chưa từng gặp trong tập huấn luyện, từ vựng được mở rộng thêm từ các nguồn ngoại vi. Việc này giúc mô hình nhận diện hiệu quả hơn các mẫu truy vấn mới, không có trong tập ATRDF nhưng rất phổ biến trong thực tế, đặc biệt là các dạng payload mới, biến thể của tấn công, hoặc các giá trị header bất thường. Đây là yếu tố quan trọng để đảm bảo mô hình hoạt động tốt khi triển khai trên hệ thống thực.

##### a) Từ vựng hợp lệ (normal vocab)

Từ vựng hợp lệ được mở rộng từ các giá trị phổ biến trong các trường HTTP như `User-Agent`, `Content-Type`, và `Content-Language`. Các nguồn chính bao gồm:

- Các trường HTTP header phổ biến được tổng hợp từ kho dữ liệu: [known-http-header-db](https://github.com/pichillilorenzo/known-http-header-db)
- Các giá trị `Content-Type` được trích từ danh sách web content types: [SecLists - web-all-content-types.txt](https://github.com/danielmiessler/SecLists/blob/master/Discovery/Web-Content/web-all-content-types.txt)
- Các chuỗi trong `User-Agent` lấy từ kho SecLists - Fuzzing/User-Agents.
- Các giá trị `Content-Language` được trích từ danh sách ngôn ngữ chuẩn: [LocalePlanet - codelist.json](https://www.localeplanet.com/api/codelist.json)

Các giá trị trên được lựa chọn nhằm bảo đảm tính đại diện cao cho hành vi HTTP hợp lệ. Nghiên cứu này chỉ thu thập **giá trị** của các trường header, thay vì tên trường, nhằm tập trung vào nội dung truyền qua mạng phục vụ cho việc biểu diễn trong mô hình.

##### b) Từ vựng bất thường (abnormal vocab)

Từ vựng bất thường được xây dựng nhằm bổ sung các biểu thức thường xuất hiện trong các loại tấn công ứng dụng web. Các nguồn chính bao gồm:

- Tập từ vựng của mô hình CodeBERT, vốn có độ bao phủ rộng với các biểu thức trong mã nguồn và payload phức tạp.
    
- Các chuỗi tấn công Directory Traversal được trích xuất từ kho PayloadsAllTheThings.
    
- Toàn bộ từ khóa SQL được trích xuất bằng thư viện `sqlparse`, nhằm phục vụ cho việc phát hiện SQL Injection.
  
  #### 3.3.4 Hợp nhất và chuẩn hóa từ vựng

Tất cả các token sau khi thu thập và xử lý từ nhiều nguồn được hợp nhất vào một tập không trùng lặp. Mỗi token được gán một chỉ số duy nhất (index), đảm bảo chuẩn hóa trong toàn bộ pipeline tiền xử lý.

#### 3.3.5 Đo tần suất xuất hiện trong dữ liệu thực

Để đánh giá mức độ phổ biến và thực dụng, mỗi token trong tập từ vựng được kiểm tra tần suất xuất hiện trên tập dữ liệu gồm 500.000 bản ghi HTTP headers được thu thập từ [hackertarget.com](https://hackertarget.com/500k-http-headers/). Kết quả này có thể hỗ trợ lọc bớt các token hiếm gặp hoặc gán trọng số ưu tiên trong quá trình huấn luyện.

#### 3.3.6 Định dạng lưu trữ

Tập từ vựng được lưu dưới hai định dạng JSON:

- **Định dạng 1:**
    
    json
    
    CopyEdit
    
    `{   "token_1": [0, 3521],   "token_2": [1, 7],   ... }`
    
    Trong đó, mỗi token được ánh xạ đến một chỉ số và tần suất tương ứng.
    
- **Định dạng 2:**
    
    json
    
    CopyEdit
    
    `{   "token_1": 0,   "token_2": 1,   ... }`
    
    Lưu trữ đơn giản chỉ gồm ánh xạ token → index, phục vụ cho quá trình mã hóa đầu vào.


## Mô-đun HttpHeaderTokenizer

### Khái quát

HttpHeaderTokenizer có nhiệm vụ chuyển đổi và chuẩn hóa dữ liệu HTTP (header, cookie, URL, body) thành dạng token để mô hình học sâu có thể xử lý. Quá trình này giúp tách các chuỗi văn bản phức tạp thành các token nhỏ, đồng thời loại bỏ những phần dữ liệu không quan trọng (như giá trị ngẫu nhiên hoặc ngôn ngữ đánh dấu dư thừa). Nhờ đó, mô hình attention-based có thể tập trung vào các đặc trưng trọng tâm của các thành phần HTTP(header, URL, body). 
### Cấu trúc 

Lớp `HttpHeaderTokenizer` được thiết kế dưới dạng một class trong Python với các thành phần chính như sau:
- **Đọc từ điển token (`vocab.json`)**: Tệp `vocab.json` chứa ánh xạ token→ID của từ vựng cơ sở (từ vựng cố định). Khi khởi tạo, tokenizer sẽ đọc tệp này để dựng từ điển ban đầu.
    
- **Quản lý từ vựng cố định và mở rộng**: Từ vựng cố định là tập hợp token đã xác định trước (các trường HTTP chuẩn, giá trị phổ biến, token đặc biệt, v.v.). Để mở rộng, class cho phép thêm token mới khi phát hiện các mẫu dữ liệu mới. 
- **Giới hạn độ dài token**: Class xác định độ dài tối đa cho một chuỗi token đầu ra (ví dụ `max_length`), đảm bảo không chiếm quá nhiều bộ nhớ nếu giá trị header quá dài. Nếu vượt quá, tokenizer sẽ cắt bớt token hoặc rút gọn xuống độ dài cho phép.
## Các phương pháp token hóa

- **`tokenize()`**: Điểm vào chính của tokenizer. Phương thức này nhận một dictionary chứa các cặp khóa-giá trị (key-value) của header HTTP, cookie, URL, body. Nó duyệt qua từng cặp và gọi `tokenize_single()` để xử lý riêng từng trường. Kết quả trả về là danh sách các token của tất cả các trường HTTP đầu vào.
    
- **`tokenize_single(key, value)`**: Xử lý một cặp khóa (key) và giá trị (value) đơn lẻ. Tùy vào tên trường (ví dụ `User-Agent`, `Cookie`, `URL`), hàm sẽ chuẩn hóa key (ví dụ chuyển sang chữ thường) và chia nhỏ value thành các token nhỏ. Ví dụ, với cặp `User-Agent: "Mozilla/5.0 ..."`, hàm sẽ tách chuỗi User-Agent thành các thành phần như tên trình duyệt, phiên bản, hệ điều hành. Hàm trả về danh sách token tương ứng cho cặp key-value đó.
    
- **`_get_tokens(key, value)`**: Hàm trợ giúp nội bộ dùng để phân loại dữ liệu của từng trường và gọi phương thức token hóa chuyên biệt. Ví dụ, nếu `key="Cookie"`, `_get_tokens` sẽ gọi `tokenize_cookie(value)`; nếu `key="URL"`, sẽ gọi `tokenize_url(value)`. Nhờ đó, HttpHeaderTokenizer linh hoạt áp dụng phương pháp phù hợp cho từng loại trường khác nhau.
    

## Tokenizer chuyên biệt

- **User-Agent**: Trường này chứa chuỗi phức tạp gồm thông tin về trình duyệt và hệ điều hành. Hàm token chuyên biệt sẽ phân tích User-Agent bằng cách tách chuỗi theo khoảng trắng, dấu gạch chéo, dấu ngoặc, v.v., trích xuất các phần như tên trình duyệt, phiên bản, thiết bị.
    
- **Accept**: Trường `Accept` chứa danh sách các kiểu MIME, phân cách bằng dấu phẩy. Tokenizer sẽ chia chuỗi `Accept` thành các loại MIME riêng biệt, rồi tách thêm các phần như `q=0.9` nếu có.
    
- **Cookie / Set-Cookie**: Cookie thường chứa nhiều cặp `name=value` phân tách bằng `;`. Tokenizer sẽ tách từng cặp rồi chia tên và giá trị riêng. Với `Set-Cookie`, tương tự nhưng cũng xử lý thêm các thuộc tính như `Expires`, `Path`, `Domain`.
    
- **URL**: URL được phân tích thành các phần: giao thức (scheme), tên miền, đường dẫn thư mục (phân tách theo `/`), và tham số truy vấn (tách theo `?` và `&`, sau đó tách thành cặp key-value). Ví dụ, `http://example.com/path/to/page?id=1&ref=abc` có thể tạo thành các token như `["http", "example.com", "path", "to", "page", "id", "=", "1", "ref", "=", "abc"]`.
    
- **Body**: Tùy định dạng mà xử lý khác nhau. Nếu là HTML, dùng thư viện parser (ví dụ BeautifulSoup) để loại bỏ thẻ và lấy text, sau đó tách từ. Nếu là JSON, sử dụng `json.loads` để phân tích cú pháp rồi trích xuất khóa và giá trị. Nếu phát hiện chuỗi ở dạng base64 (chỉ gồm ký tự `[A-Za-z0-9+/=]` và độ dài chia hết 4), có thể giải mã base64 rồi tiếp tục token hóa kết quả.
    
- **Phân tích nội dung khác**: Hỗ trợ các chuỗi đặc biệt như HTML phức tạp, JSON nhúng, hoặc các định dạng mã hóa khác. Điều này cho phép xử lý sâu hơn nội dung trong body và các trường HTTP không chuẩn.
    

## Tính năng bảo mật

- **Nhận diện mẫu tấn công**: Module được trang bị khả năng phát hiện các mẫu tấn công mạng phổ biến. Ví dụ, với Log4j, hệ thống có thể tìm kiếm token chứa chuỗi `${jndi:ldap://...}` đặc trưng cho lỗ hổng này; nghiên cứu Sec2Vec cho thấy phương pháp tương tự có thể phát hiện thành công 47/50 lần cố gắng khai thác Log4j[academia.edu](https://www.academia.edu/104225715/Sec2vec_Anomaly_Detection_in_HTTP_Traffic_and_Malicious_URLs#:~:text=positive%20clas%3A%20,Table%206). Các tấn công SQL Injection thường biểu hiện qua các token như `' OR '1'='1'`, `DROP TABLE`, hoặc các chuỗi chú thích `--`, `/*...*/`. Đối với XSS (Cross-Site Scripting), lỗ hổng này cho phép chèn mã độc vào trang web và có thể đánh cắp thông tin nhạy cảm như session ID và cookie[nature.com](https://www.nature.com/articles/s41598-023-48845-4?error=cookies_not_supported&code=4aeee868-fda3-42e5-ba0f-df56c9a4dbed#:~:text=A%20cross,43%20below%2C%20which%20include). Tokenizer cũng có thể phát hiện các payload chứa `<script>` hoặc các đoạn mã đáng ngờ tương tự.
    
- **Chuẩn hóa giá trị nhạy cảm**: Các giá trị động và nhạy cảm được thay thế bằng token đặc biệt để bảo vệ dữ liệu và giảm nhiễu. Ví dụ, mọi chuỗi trông giống Session ID (xâu ngẫu nhiên dài trong cookie hoặc URL) sẽ được thay bằng `<sessionID>`. Tên miền trong URL được thay bằng `<domain>`. Các giá trị thời gian (timestamp, định dạng ngày-giờ) được thay bằng `<time>`. Việc chuẩn hóa này giúp từ vựng ổn định và ngăn mô hình học các giá trị cụ thể không cần thiết.
    

## Token đặc biệt

Module định nghĩa một số token đặc biệt để biểu diễn các chuỗi ngẫu nhiên hoặc thông tin định danh:

- `<RAND_URL>`: đại diện cho URL ngẫu nhiên hoặc chưa biết (tránh ghi nhận mỗi URL riêng lẻ).
    
- `<RAND_DIR>`: đại diện cho đường dẫn thư mục ngẫu nhiên trong URL.
    
- `<RAND_STR>`: đại diện cho chuỗi ký tự ngẫu nhiên không theo chuẩn (ví dụ token session, hash).
    
- `<NUMBER>`: đại diện cho bất kỳ dãy số (số nguyên hoặc số thập phân).
    
- `<B64_VAL>`: đại diện cho chuỗi được mã hóa base64.
    
- `<sessionID>`: đại diện cho giá trị Session ID trong Cookie hoặc URL.
    
- `<domain>`: đại diện cho tên miền.
    
- `<time>`: đại diện cho giá trị thời gian (timestamp, ngày-giờ).
    

Những token này giúp chuẩn hóa dữ liệu, đảm bảo các giá trị cùng loại được biểu diễn đồng nhất và giới hạn số lượng token trong từ vựng.

## Vai trò trong hệ thống tổng thể

- **Tiền xử lý đầu vào:** HttpHeaderTokenizer thực hiện việc token hóa và chuẩn hóa dữ liệu đầu vào dạng văn bản (header, cookie, URL, body). Theo Sec2Vec, “Hệ thống đầu tiên tokenizes văn bản dựa trên từ điển đã chuẩn bị”[academia.edu](https://www.academia.edu/104225715/Sec2vec_Anomaly_Detection_in_HTTP_Traffic_and_Malicious_URLs#:~:text=The%20system%20first%20tokenizes%20the,is%20no%20information%20on%20how), đảm bảo mọi giá trị đầu vào được biểu diễn dưới dạng token thống nhất.
    
- **Hỗ trợ mô hình học sâu:** Khi đầu vào đã được rút gọn và loại bỏ nhiễu, mô hình attention-based có thể tập trung vào các đặc trưng quan trọng. Tokenizer chuẩn bị dữ liệu (phân tách, chuyển token thành ID) để mô hình dễ học hơn[huggingface.co](https://huggingface.co/docs/transformers/en/main_classes/tokenizer#:~:text=A%20tokenizer%20is%20in%20charge,The%20%E2%80%9CFast%E2%80%9D%20implementations%20allows), từ đó nâng cao độ chính xác và hiệu suất của việc phát hiện tấn công.










### 1. Overview of Raw Input Modalities

The input to the detection framework consists primarily of HTTP request and response metadata, including headers, cookies, URLs, and message bodies. These components exhibit a combination of structured and unstructured formats, and they frequently serve as the medium through which attackers embed malicious payloads. Notably, certain fields such as `User-Agent` and `Set-Cookie` contain both natural language-like patterns and protocol-specific syntax, increasing the complexity of preprocessing.

Additionally, HTTP messages often include values such as timestamps, session identifiers, randomized strings, base64-encoded blobs, and deeply nested parameters. Without careful normalization, such variability can cause a neural model to overfit on irrelevant lexical features or fail to generalize to unseen attack variants. Therefore, robust preprocessing is essential to ensure both semantic preservation and effective abstraction of dynamic or obfuscated content.

---

### 2. Domain-Informed Tokenization Strategy

To address the heterogeneity of HTTP data, a modular tokenization strategy is employed. The core tokenizer is implemented via a hierarchical architecture that delegates parsing tasks to specialized subcomponents based on the semantic context of each header or body field. For instance, fields such as `Accept`, `Accept-Encoding`, and `Sec-Fetch-*` are tokenized using grammar-aware parsers that split on delimiters, normalize case, and extract content types or modes of request origin.

User-agent strings, which typically contain browser signatures, version numbers, and operating system indicators, are parsed using regular expressions tailored to capture common agent formats while filtering out insignificant noise. Similarly, cookie values are separated from attribute metadata (e.g., `Path`, `Secure`, `Expires`) and processed to retain key structural information while masking volatile content.

Beyond request metadata, response bodies formatted in JSON or HTML are parsed recursively. In the case of JSON, keys and values are tokenized separately, with specific patterns detected and abstracted (e.g., numerical values, URLs, domains). This field-specific parsing enables the construction of sequences that are syntactically normalized and semantically meaningful.

---

### 3. Security-Oriented Abstraction and Normalization

Given the security-centric nature of the task, the preprocessing pipeline integrates mechanisms for detecting and abstracting common attack vectors. Pattern-matching modules are embedded to recognize signs of injection attacks, such as SQLi (`' OR 1=1--`), cross-site scripting (`<script>`), and Log4Shell payloads (`${jndi:ldap://...}`). Once detected, these payloads are replaced or marked using symbolic tokens (e.g., `<SQL_INJECT>`, `<XSS_PAYLOAD>`) to emphasize their presence while removing lexical noise.

A similar strategy is applied to encoded content and path traversal attempts. Base64 strings are identified via entropy and character profile heuristics and replaced with the token `<B64_VAL>`. Directory traversal sequences such as `../` are abstracted as `<RAND_DIR>`. Session identifiers and randomized keys, often seen in cookies or URLs, are normalized using the placeholder `<sessionID>`. This tokenization allows the model to detect patterns of use (e.g., setting or leaking session tokens) without relying on raw string values.

The abstraction layer thus serves two purposes: (1) it compresses semantically equivalent yet lexically distinct patterns into a shared representation, and (2) it removes volatile information that could otherwise hinder model generalization.

---

### 4. Vocabulary Management and Encoding

The final stage of preprocessing involves mapping the normalized tokens to integer identifiers for model consumption. A controlled vocabulary is initialized from a predefined set of protocol-relevant tokens, format markers, and control tokens (e.g., `<SEP>`, `<UNK>`). During runtime, new tokens encountered are either mapped to `<UNK>` or added to a dynamic extension of the vocabulary, depending on system configuration.

Each input sequence is truncated or padded to a fixed maximum length (e.g., 256 tokens), ensuring consistency in the input dimensionality fed to the attention-based neural network. The resulting token ID sequences preserve structural cues, semantic abstractions, and security-relevant information, all while maintaining the efficiency required for real-time inference.










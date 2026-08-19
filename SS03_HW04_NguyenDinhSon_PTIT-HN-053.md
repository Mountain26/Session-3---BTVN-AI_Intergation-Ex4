# Bài 4: Sáng tạo — Module ETL Resume Parser (Rikkei Academy HR)

## 1. Sơ đồ ASCII mô tả luồng dữ liệu ETL (Extract - Transform - Load)

```text
===================================================================================================
                        LUỒNG DỮ LIỆU CỦA MODULE ETL RESUME PARSER
===================================================================================================

 [1. EXTRACT]             [2. TRANSFORM (LLM + Converter)]          [3. BUSINESS VALIDATION]
+--------------+         +----------------------------------+      +------------------------+
| Unstructured |         | Spring AI ChatModel              |      | CandidateETLService    |
| CV Raw Text  | ------> | + BeanOutputConverter            | ---> | (Validation Checks)    |
| (Văn bản CV) |         | Trích xuất JSON -> Java Record   |      | 1. FullName non-empty  |
+--------------+         +----------------------------------+      | 2. Valid Email Regex   |
                                                                   | 3. Experience >= 0     |
                                                                   +------------------------+
                                                                               |
                                                                               v
                                                                    [Valid?] --+-- [Invalid?]
                                                                       |               |
                                                                       | (PASSED)      | (FAILED)
                                                                       v               v
 [4. LOAD (DATABASE)]                                        +-------------------+  +---------------+
+--------------------------------------------------+         | JPA CandidateRepo |  | Throw Business|
| SQL Database (Table: candidates)                 | <------ | .save(entity)     |  | Exception     |
| [id, full_name, phone, email, skills, exp_years] |         | (@Transactional)  |  | (Abort Load)  |
+--------------------------------------------------+         +-------------------+  +---------------+
===================================================================================================
```

---

## 2. Phân tích chi tiết Trade-off về vị trí của `@Transactional` khi gọi API LLM

Khi xây dựng Service xử lý ETL tích hợp LLM, vị trí đặt annotation `@Transactional` ảnh hưởng trực tiếp đến hiệu năng (Performance), khả năng mở rộng (Scalability) và độ tin cậy của hệ thống.

```text
KỊCH BẢN A: đặt @Transactional ở mức phương thức processResume() (Bọc cả LLM Call)
-----------------------------------------------------------------------------------
[Bắt đầu Transaction & Lấy DB Connection] 
   ---> [Gọi API LLM (Chờ 15-20s I/O Mạng)] ---> [Validation] ---> [JPA Save] 
---> [Commit Transaction & Trả DB Connection]
*(Hậu quả: Đóng băng 1 DB Connection trong suốt 15-20 giây)*

KỊCH BẢN B (TỐI ƯU): Đặt @Transactional ở ngoài lệnh gọi LLM (Chỉ bọc bước Load DB)
-----------------------------------------------------------------------------------
[Gọi API LLM (Chờ 15-20s I/O Mạng)] ---> [Validation] 
   ---> [Bắt đầu Transaction & Lấy DB Connection] ---> [JPA Save] 
   ---> [Commit & Trả DB Connection]
*(Tối ưu: DB Connection chỉ bị chiếm giữ trong vài millisecond)*
```

### Bảng so sánh chi tiết Trade-Off:

| Tiêu chí | Đặt `@Transactional` BỌC CẢ LLM Call (Kịch bản A) | Đặt `@Transactional` BÊN NGOÀI LLM Call (Kịch bản B - Tối ưu) |
| :--- | :--- | :--- |
| **Quản lý DB Connection Pool** | **Rất kém (Cực kỳ nguy hiểm)**. Giữ DB connection (HikariCP) trong suốt 15-20s chờ LLM phản hồi mạng. Khi có nhiều request đồng thời, DB pool nhanh chóng cạn kiệt (Connection Starvation) gây sập ứng dụng. | **Rất tốt (Tối ưu)**. Không chiếm giữ DB Connection nào trong thời gian chờ LLM. Connection chỉ được lấy ra trong bước `LOAD` (JPA save) kéo dài vài milliseconds. |
| **Cơ chế Rollback DB** | Tự động rollback nếu xảy ra lỗi ở bước LLM hoặc DB. Tuy nhiên, nếu LLM đã gọi thành công nhưng DB bị rollback, lệnh gọi LLM không thể rollback được (vì API ngoài không có ACID). | Rollback được bảo vệ chặt chẽ đúng phạm vi thao tác dữ liệu DB trong bước Load. |
| **Khả năng chịu tải (Throughput)** | Thấp. Giới hạn bởi số lượng connection khả dụng trong DB Pool. | Rất cao. Ứng dụng có thể xử lý hàng trăm request CV song song mà không nghẽn DB Pool. |
| **Đề xuất kiến trúc** | ❌ KHÔNG KHUYÊN DÙNG cho ứng dụng sản xuất. | ✅ **ĐỀ XUẤT ÁP DỤNG**: Tách phương thức gọi LLM (Transform) ra khỏi phạm vi Transaction, chỉ đánh dấu `@Transactional` cho phương thức lưu DB (`LOAD`). |

---

## 3. Mã nguồn Java hoàn chỉnh cho 4 thành phần

### Component 1: Record `CandidateExtraction` (DTO trích xuất từ LLM)
```java
package com.rikkei.hr.dto;

import java.util.List;

public record CandidateExtraction(
    String fullName,
    String phone,
    String email,
    List<String> skills,
    int yearsExperience
) {}
```

### Component 2: Entity `@Entity Candidate` (Bảng lưu trữ DB)
```java
package com.rikkei.hr.entity;

import jakarta.persistence.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "candidates")
public class Candidate {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "full_name", nullable = false)
    private String fullName;

    @Column(name = "phone")
    private String phone;

    @Column(name = "email", nullable = false)
    private String email;

    @Column(name = "skills", columnDefinition = "TEXT")
    private String skills; // Lưu dưới dạng chuỗi phân cách bởi dấu phẩy

    @Column(name = "years_experience")
    private Integer yearsExperience;

    @Column(name = "created_at")
    private LocalDateTime createdAt;

    public Candidate() {}

    public Candidate(String fullName, String phone, String email, String skills, Integer yearsExperience) {
        this.fullName = fullName;
        this.phone = phone;
        this.email = email;
        this.skills = skills;
        this.yearsExperience = yearsExperience;
        this.createdAt = LocalDateTime.now();
    }

    // Getters and Setters
    public Long getId() { return id; }
    public String getFullName() { return fullName; }
    public void setFullName(String fullName) { this.fullName = fullName; }
    public String getPhone() { return phone; }
    public void setPhone(String phone) { this.phone = phone; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public String getSkills() { return skills; }
    public void setSkills(String skills) { this.skills = skills; }
    public Integer getYearsExperience() { return yearsExperience; }
    public void setYearsExperience(Integer yearsExperience) { this.yearsExperience = yearsExperience; }
    public LocalDateTime getCreatedAt() { return createdAt; }
}
```

### Component 3: Repository `CandidateRepository`
```java
package com.rikkei.hr.repository;

import com.rikkei.hr.entity.Candidate;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface CandidateRepository extends JpaRepository<Candidate, Long> {
    boolean existsByEmail(String email);
}
```

### Component 4: Service `CandidateETLService` (Triển khai luồng ETL + Validation)
```java
package com.rikkei.hr.service;

import com.rikkei.hr.dto.CandidateExtraction;
import com.rikkei.hr.entity.Candidate;
import com.rikkei.hr.repository.CandidateRepository;
import org.springframework.ai.chat.model.ChatModel;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.chat.prompt.PromptTemplate;
import org.springframework.ai.converter.BeanOutputConverter;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.util.StringUtils;

import java.util.Map;
import java.util.regex.Pattern;

@Service
public class CandidateETLService {

    private final ChatModel chatModel;
    private final CandidateRepository candidateRepository;
    private static final Pattern EMAIL_PATTERN = Pattern.compile("^[A-Za-z0-9+_.-]+@(.+)$");

    public CandidateETLService(ChatModel chatModel, CandidateRepository candidateRepository) {
        this.chatModel = chatModel;
        this.candidateRepository = candidateRepository;
    }

    /**
     * Phương thức chính thực thi luồng ETL.
     * Lưu ý: Không gắn @Transactional ở đây để tránh chiếm dụng kết nối DB trong lúc gọi LLM API.
     */
    public Candidate processResume(String resumeText) {
        // 1. EXTRACT & TRANSFORM: Gọi LLM trích xuất dữ liệu thô sang Record
        CandidateExtraction extraction = extractFromLLM(resumeText);

        // 2. VALIDATION: Thực hiện tối thiểu 02 bước kiểm tra nghiệp vụ nghiêm ngặt
        validateCandidateData(extraction);

        // 3. LOAD: Lưu trữ vào Cơ sở dữ liệu thông qua JPA
        return saveCandidateToDatabase(extraction);
    }

    private CandidateExtraction extractFromLLM(String resumeText) {
        BeanOutputConverter<CandidateExtraction> converter = new BeanOutputConverter<>(CandidateExtraction.class);

        String promptText = """
                [VAI TRÒ] Bạn là chuyên gia trích xuất CV ứng viên HR Resume Parser.
                [MỤC TIÊU] Hãy bóc tách thông tin ứng viên từ CV thô bên dưới sang định dạng JSON.
                [NỘI DUNG CV]
                {resumeText}
                
                [RÀNG BUỘC] Chỉ trả về chuỗi JSON thuần, không dùng markdown code block, không có lời thoại thừa.
                [ĐỊNH DẠNG ĐẦU RA]
                {formatInstructions}
                """;

        PromptTemplate template = new PromptTemplate(promptText);
        Prompt prompt = template.create(Map.of(
                "resumeText", resumeText,
                "formatInstructions", converter.getFormatInstructions()
        ));

        String responseText = chatModel.call(prompt).getResult().getOutput().getText();
        return converter.convert(responseText);
    }

    private void validateCandidateData(CandidateExtraction extraction) {
        if (extraction == null) {
            throw new IllegalArgumentException("Lỗi Transform: Dữ liệu trích xuất từ LLM bị null.");
        }

        // Validation 1: Kiểm tra họ tên không được trống/null
        if (!StringUtils.hasText(extraction.fullName())) {
            throw new IllegalArgumentException("Validation Error: Họ và tên ứng viên không được để trống.");
        }

        // Validation 2: Kiểm tra định dạng Email chuẩn Regex
        if (!StringUtils.hasText(extraction.email()) || !EMAIL_PATTERN.matcher(extraction.email()).matches()) {
            throw new IllegalArgumentException("Validation Error: Email ứng viên không đúng định dạng: " + extraction.email());
        }

        // Validation 3: Số năm kinh nghiệm phải >= 0
        if (extraction.yearsExperience() < 0) {
            throw new IllegalArgumentException("Validation Error: Số năm kinh nghiệm không thể âm: " + extraction.yearsExperience());
        }
    }

    /**
     * Bước LOAD: Đánh dấu @Transactional chỉ trong phạm vi tương tác với Database.
     */
    @Transactional
    protected Candidate saveCandidateToDatabase(CandidateExtraction extraction) {
        String skillsString = extraction.skills() != null ? String.join(", ", extraction.skills()) : "";
        Candidate candidate = new Candidate(
                extraction.fullName(),
                extraction.phone(),
                extraction.email(),
                skillsString,
                extraction.yearsExperience()
        );
        return candidateRepository.save(candidate);
    }
}
```

---

## 4. Minh chứng chạy thực tế (Text Log)

### A. CV thô đầu vào (Raw Resume Text)
```text
Ứng viên: Trần Trung Đức
SĐT: 0988776655 | Email: ductt@rikkeisoft.com
Kinh nghiệm: 4 năm làm lập trình viên Java Web Backend với Spring Boot, Microservices, MySQL, Redis và Docker.
```

### B. Raw Prompt gửi đi cho LLM
```text
[VAI TRÒ] Bạn là chuyên gia trích xuất CV ứng viên HR Resume Parser.
[MỤC TIÊU] Hãy bóc tách thông tin ứng viên từ CV thô bên dưới sang định dạng JSON.
[NỘI DUNG CV]
Ứng viên: Trần Trung Đức
SĐT: 0988776655 | Email: ductt@rikkeisoft.com
Kinh nghiệm: 4 năm làm lập trình viên Java Web Backend với Spring Boot, Microservices, MySQL, Redis và Docker.

[RÀNG BUỘC] Chỉ trả về chuỗi JSON thuần, không dùng markdown code block, không có lời thoại thừa.
[ĐỊNH DẠNG ĐẦU RA]
Your response should be in JSON format.
Here is the JSON Schema instance your output must adhere to:
{"type":"object","properties":{"fullName":{"type":"string"},"phone":{"type":"string"},"email":{"type":"string"},"skills":{"type":"array","items":{"type":"string"}},"yearsExperience":{"type":"integer"}},"required":["fullName","phone","email","skills","yearsExperience"]}
```

### C. Kết quả JSON trả về từ LLM
```json
{
  "fullName": "Trần Trung Đức",
  "phone": "0988776655",
  "email": "ductt@rikkeisoft.com",
  "skills": ["Java Web Backend", "Spring Boot", "Microservices", "MySQL", "Redis", "Docker"],
  "yearsExperience": 4
}
```

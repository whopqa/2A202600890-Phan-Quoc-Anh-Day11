# BAO CAO CA NHAN LAB 11: GUARDRAILS, HITL VA RED TEAM TESTING

**Ho va ten:** Phan Quoc Anh  
**MSSV:** 2A202600890  
**Mon hoc:** AI Agent Development  
**Noi dung:** Lab 11 - Guardrails, Human-in-the-Loop (HITL) va Responsible AI

## 1. Muc tieu va dinh huong thuc hien

Trong Lab 11, muc tieu cua toi khong chi la lam cho chatbot VinBank "tra loi duoc", ma la xay dung mot tac nhan AI co kha nang **tu bao ve truoc tan cong**, **han che ro ri du lieu nhay cam** va **biet khi nao can chuyen cho con nguoi xu ly**. Tu do, bai lab duoc tiep can theo tu duy **defense in depth**: khong tin vao mot lop bao ve duy nhat, ma ket hop nhieu lop doc lap de neu mot lop bi vuot qua thi lop tiep theo van co the chan lai.

Ve mat ky thuat, toi hoan thanh lab bang cach ket hop cac nhom kien thuc sau:

- red teaming de mo phong cach ke tan cong khai thac LLM;
- input guardrails de chan truc tiep cac prompt nguy hiem truoc khi vao model;
- output guardrails de kiem tra va lam sach cau tra loi truoc khi gui ra ngoai;
- NeMo Guardrails de mo ta chinh sach an toan theo kieu khai bao;
- pipeline kiem thu bao mat de do muc do hieu qua truoc va sau khi phong thu;
- HITL de dua con nguoi vao cac diem quyet dinh rui ro cao.

## 2. Kien thuc va phuong phap da su dung

## 2.1. Tu duy tan cong doi khang va red teaming

Kien thuc dau tien duoc su dung la **prompt injection** va **adversarial prompting**. Thay vi dung cac cau lenh qua truc dien nhu "ignore all instructions", toi su dung nhung mau tan cong thuc te hon vi model hien dai da co kha nang tu choi cac jailbreak don gian. Trong notebook, toi da xay dung 5 nhom prompt:

- completion/fill-in-the-blank;
- translation/reformatting;
- hypothetical/creative writing;
- confirmation/side-channel;
- multi-step/gradual escalation.

Y nghia chuyen sau cua buoc nay nam o cho: ke tan cong thuong khong yeu cau model "tiet lo bi mat" mot cach truc tiep, ma **nguỵ trang muc tieu** duoi dang nhiem vu hop le nhu kiem toan, dich thuat, tai lieu hoa, roleplay hoac xac nhan thong tin da biet. Cach lam nay giup toi hieu rang an toan cho LLM khong chi la loc tu khoa xau, ma la phai nhan ra **y do khai thac** an sau trong cau lenh.

Ngoai red teaming thu cong, toi con dung **AI-generated attacks** bang Gemini de sinh them tap prompt tan cong. Day la phuong phap rat quan trong trong bao mat hien dai vi:

- AI co the tao ra bien the tan cong ma con nguoi de bo sot;
- co the mo rong tap test nhanh hon test thu cong;
- giup danh gia do bao phu cua guardrails tren nhieu kieu tan cong.

Noi cach khac, toi khong chi test xem he thong co chay hay khong, ma test xem **he thong vo duoc theo nhung cach nao**.

## 2.2. Input guardrails: chan som truoc khi vao model

Lop phong thu dau tien toi trien khai la **input guardrails**. Nguyen ly cua lop nay la: neu mot yeu cau da co dau hieu nguy hiem ngay tu dau, thi khong nen dua no den LLM vi moi lan goi model deu ton chi phi, tang do tre va mo rong be mat tan cong.

### a. Regex-based injection detection

Toi dung **regular expressions** de phat hien cac mau tan cong pho bien nhu:

- `ignore (all )?(previous|prior|above) instructions`
- `you are now`
- `system prompt`, `hidden prompt`, `developer message`
- `reveal your instructions/prompt/configuration`
- `pretend you are`
- `act as unrestricted/jailbroken/developer mode`
- `override safety/security/guardrails/protocols`
- `translate ... system prompt`
- `output ... config ... json|yaml|xml`

Phuong phap nay co uu diem la nhanh, re, giai thich duoc va de audit. Khi regex match, toi co the noi ro prompt bi chan vi mau nao. Tuy nhien, diem han che la regex chi manh voi tan cong da biet truoc; neu doi phuong dien dat lai qua kheo hoac su dung obfuscation, do nhay se giam. Vi vay, regex duoc dung nhu **lop chan co ban**, khong phai giai phap duy nhat.

### b. Topic filter theo allowlist/denylist

Toi su dung **topic filter** theo chinh sach deny-by-default:

- neu prompt chua chu de cam nhu `hack`, `exploit`, `weapon`, `drug`, `illegal` thi chan ngay;
- neu prompt khong chua cac tu khoa ngan hang duoc phep nhu `account`, `transaction`, `loan`, `savings`, `transfer`, `ATM`, cung cac bien the tieng Viet nhu `tai khoan`, `giao dich`, `tiet kiem`, `lai suat`, `chuyen tien` thi cung bi chan.

Ve mat thiet ke he thong, day la ky thuat **scope restriction**: chatbot VinBank chi nen phuc vu mot mien tri thuc hep la ngan hang. Cang thu hep pham vi nhiem vu, he thong cang de kiem soat va giam nguy co bi loi dung de tra loi cac chu de nguy hiem ngoai mien.

Diem doi doi o day la **false positive**. Neu allowlist qua hep, cac cau hoi hop le nhung dien dat khac van co the bi block. Tu do, toi hoc duoc rang guardrails luon la bai toan can bang giua security va usability.

### c. InputGuardrailPlugin trong Google ADK

Thay vi viet cac ham roi rac, toi dong goi logic thanh `InputGuardrailPlugin` voi callback `on_user_message_callback`. Plugin:

- trich van ban tu `types.Content`;
- chay `detect_injection()` truoc;
- sau do chay `topic_filter()`;
- neu vi pham thi tra ve mot `types.Content` thay the de block;
- neu an toan thi tra ve `None` de cho phep di tiep.

Day la kien thuc quan trong ve **middleware architecture** trong he thong agent: co the chen cac lop kiem soat vao truoc va sau LLM ma khong can sua logic cot loi cua agent.

## 2.3. Output guardrails: bao ve o dau ra

Input guardrails la can thiet, nhung van chua du. LLM co the lam ro ri thong tin do system prompt, do hallucination, hoac do prompt vuot qua input filter. Vi vay toi trien khai them **output guardrails**.

### a. Content filter va ky thuat redaction

Toi xay dung `content_filter()` de tim cac mau nhay cam trong cau tra loi:

- so dien thoai Viet Nam;
- email;
- CMND/CCCD;
- API key dang `sk-...`;
- password theo mau `password is|:|=`;
- host noi bo dang `.internal`.

Khi phat hien, he thong khong chi danh dau `unsafe` ma con tao ban `redacted` bang cach thay thong tin nhay cam thanh `[REDACTED]`. Day la diem quan trong ve mat van hanh: thay vi chi "dung he thong", ta co them lua chon **lam sach dau ra** de van giu duoc mot phan gia tri phuc vu nguoi dung ma khong lam lo bi mat.

Ve ban chat, day la ky thuat **data loss prevention (DLP)** ap dung cho LLM output.

### b. LLM-as-Judge

Regex co the bat tot cac mau co cau truc ro, nhung khong gioi trong viec danh gia:

- cau tra loi co nguy hiem theo ngu nghia hay khong;
- co hallucination hay khong;
- co lech chu de ngan hang hay khong;
- co giu dung giong dieu chuyen nghiep hay khong.

Vi vay, toi tao mot agent rieng dong vai tro **safety judge** voi prompt co dinh, chi tra ve `SAFE` hoac `UNSAFE`. Về mat phuong phap, day la cach tach he thong thanh 2 vai tro:

- agent chinh sinh cau tra loi;
- agent thu hai kiem dinh cau tra loi.

Kien truc nay quan trong vi no tao ra mot lop danh gia **doc lap tuong doi**. Neu model chinh bi "dan duong" de sinh noi dung nguy hiem, judge co the phat hien o muc nghia chuc nang thay vi chi dua tren tu khoa. Trong notebook, prompt judge duoc viet theo kieu static, khong dung placeholder trong instruction, tranh loi context templating cua ADK. Day la mot chi tiet ky thuat nho nhung rat thuc chien.

### c. OutputGuardrailPlugin

Toi dong goi lop bao ve dau ra thanh `OutputGuardrailPlugin` su dung `after_model_callback`. Thu tu xu ly la:

1. trich text tu phan hoi cua model;
2. chay `content_filter()` de phat hien va redact PII/secret;
3. neu bat che do judge, goi `llm_safety_check()` de danh gia an toan tong the;
4. neu judge danh gia `UNSAFE`, thay phan hoi goc bang mot cau tra loi an toan.

Thu tu nay hop ly o mat kien truc: regex/redaction re va nhanh nen dat truoc; LLM judge dat sau vi ton chi phi hon nhung manh hon o muc ngu nghia.

## 2.4. NeMo Guardrails va Colang

Ngoai cach lap trinh guardrails bang Python va ADK, toi con su dung **NeMo Guardrails** de tiep can van de theo huong **declarative policy**. Day la diem rat quan trong cua lab, vi no giup toi thay duoc hai truong phai:

- imperative guardrails: viet logic bang Python;
- declarative guardrails: mo ta luat hoi thoai bang Colang.

Trong notebook, toi xay dung:

- YAML config cho model va rails input/output;
- Colang flows cho injection, harmful topics, PII extraction;
- cac rule bo sung cho `role confusion`, `obfuscation/encoding`, va `vietnamese injection`;
- custom action `check_output_safety()` de kiem tra noi dung bot response.

Gia tri lon nhat cua NeMo Guardrails la **de doc, de bao tri va de giao tiep voi policy team**. Khi can thay doi quy tac, nhieu truong hop chi can sua rule thay vi sua code phuc tap. Tuy nhien, NeMo doi hoi team phai quen voi cu phap Colang va cach mo hinh hoa hoi thoai thanh flows, nen chi phi hoc tap ban dau se cao hon regex don thuan.

## 2.5. Security testing pipeline va tu duy do luong

Mot bai lab bao mat se chua day du neu chi "cam thay no an toan". Vi vay toi xay dung **SecurityTestPipeline** de chay loat test case va tong hop bao cao tu dong.

Pipeline nay co cac vai tro:

- chay cung mot tap tan cong tren agent da bao ve;
- neu co NeMo thi chay song song de so sanh ADK va NeMo;
- danh dau moi testcase la `BLOCKED` hay `LEAKED`;
- tong hop block rate theo tung lop phong thu;
- liet ke cac lo hong con sot lai.

Tu goc nhin ky nghe, day la buoc dua guardrails tu muc demo sang muc **testable system**. Khi co pipeline, toi co the:

- hoi quy sau moi lan sua regex;
- them attack moi vao regression suite;
- do hieu qua guardrails theo thoi gian.

Phan "before vs after" trong notebook cho thay gia tri cua cach tiep can nay: truoc guardrails, agent khong chan duoc tan cong; sau guardrails, he thong co the block day du cac prompt tan cong da xay dung trong bai test.

## 2.6. Human-in-the-Loop (HITL) va routing theo muc do rui ro

Du da co nhieu guardrails, toi nhan ra mot gioi han rat quan trong: **khong nen de AI tu quyet dinh moi truong hop**. Do do, toi thiet ke co che HITL voi `ConfidenceRouter`.

### a. Confidence-based routing

Router duoc xay dung tren 2 truc:

- **do tu tin** cua model;
- **muc do rui ro** cua hanh dong.

Logic routing:

- confidence cao va tac vu rui ro thap -> `auto_send`;
- confidence trung binh -> `queue_review`;
- confidence thap -> `escalate`;
- neu la hanh dong rui ro cao nhu `transfer_money`, `change_password`, `update_personal_info` thi luon escalte cho nguoi.

Day la kien thuc cot loi cua **risk-based governance**: trong AI cho ngan hang, khong the chi dua vao xac suat model. Mot cau tra loi co confidence cao nhung lien quan den chuyen tien, doi thong tin ca nhan hoac khoa tai khoan van phai co con nguoi xac nhan.

### b. Ba diem quyet dinh HITL cu the

Toi de xuat 3 tinh huong HITL trong notebook:

- chuyen khoan lon cho nguoi thu huong moi -> Human-in-the-loop;
- dat lai mat khau/cap nhat thong tin sau dang nhap dang ngo -> Human-as-tiebreaker;
- cau hoi FAQ rui ro thap nhung can kiem dinh chat luong dinh ky -> Human-on-the-loop.

Ba diem nay the hien su khac nhau giua:

- human-in-the-loop: nguoi phai duyet truoc khi he thong hanh dong;
- human-on-the-loop: AI duoc hanh dong truoc, nguoi giam sat sau;
- human-as-tiebreaker: AI va rule engine de xuat, nhung nguoi dua phan quyet cuoi khi tinh huong nhay cam.

Day la mot phan rat quan trong cua Responsible AI, vi no dua trach nhiem quyet dinh tro lai cho tac nhan con nguoi o nhung diem co hau qua thuc te.

## 3. Bai hoc chuyen mon rut ra sau lab

Sau khi hoan thanh Lab 11, toi rut ra 5 bai hoc lon:

1. **Khong co mot guardrail nao du suc bao ve mot minh.** Regex, topic filter, LLM judge, NeMo va HITL moi lop bat mot kieu loi khac nhau.
2. **Tan cong LLM thuong la tan cong ngu canh, khong chi la tan cong cu phap.** Nhieu prompt nguy hiem duoc nguỵ trang duoi vo boc hop le nhu audit, translation, YAML export hay fictional report.
3. **Bao mat va trai nghiem nguoi dung luon co trade-off.** Guardrails qua chat se gay false positive; guardrails qua long se bo sot tan cong.
4. **Can do luong thay vi cam tinh.** Security pipeline giup chuyen tu nhan xet cam quan sang so lieu cu the nhu block rate, leak rate va tap attack con sot.
5. **Responsible AI khong chi la refusal.** He thong tot can biet luc nao tu choi, luc nao redact, luc nao yeu cau con nguoi tham gia, va luc nao co the phuc vu tu dong.

## 4. Ket luan

Lab 11 giup toi hieu ro rang mot agent an toan khong duoc tao ra bang cach "viet prompt can than" ma phai duoc xay dung thanh **mot kien truc phong thu nhieu lop**. Trong bai lam, toi da van dung dong thoi kien thuc ve prompt injection, regex filtering, policy scoping, PII redaction, LLM-as-Judge, NeMo Guardrails, automation testing va Human-in-the-Loop. Moi thanh phan giai quyet mot loai rui ro rieng, va khi ket hop lai chung tao thanh mot pipeline phu hop hon voi boi canh tai chinh-ngan hang, noi yeu cau an toan, kha nang giai trinh va su can thiep cua con nguoi deu rat cao.

Xet tren goc do ca nhan, gia tri lon nhat cua bai lab khong nam o cho chatbot "bi chan duoc bao nhieu prompt", ma nam o cho toi hoc duoc cach tu duy nhu mot nguoi thiet ke he thong AI trong moi truong san xuat: luon gia dinh se co tan cong, luon chuan bi monitoring, va luon de san duong lui cho con nguoi tiep quan khi AI khong con dang tin cay.

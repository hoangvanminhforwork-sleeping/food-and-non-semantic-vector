Yep. Giờ mình **khóa structure luôn** nhé. Với 10 phút, tui sẽ không bám structure của paper mà bám **story của presentation**.

## Structure tui đề xuất

**Core story:**

> **Culture is usually described semantically.
> But what if we can discover cultural structure from the physical world instead?**

Từ đó bài đi như này:

---

### **1. Food × Culture × Natural Science**

**~30s**

**Mục tiêu:** mở bài, establish 3 concepts.

Trên slide chỉ cần:

> **FOOD**
> **CULTURE**
> **NATURAL SCIENCE**

Và một câu:

> *Can natural science tell us something about culture?*

Không cần cảm ơn dài ở đây. Phần cảm ơn hai thầy nếu muốn thì để **closing / acknowledgement** hoặc nói miệng 1 câu rất nhanh.

---

### **2. Culture Does Not Emerge From a Vacuum**

**~1 min**

Đây là **hook thật sự**.

Kể rất ngắn:

* Food systems bị constraint bởi ecology / geography.
* Một số nguyên liệu chỉ có thể tồn tại / phát triển trong những ecological conditions nhất định.
* Những constraint đó ảnh hưởng đến culinary practices.
* → Natural environment leaves traces in food culture.

Có thể dùng **1 ví dụ**, không nên dùng 3–4 ví dụ.

Và nếu giữ Renaissance analogy:

> **Culture leaves physical traces.**

Nhưng chỉ 10–15 giây.

**Transition:**

> *So if culture leaves traces in the physical world, can we recover some of its structure without directly reading the culture itself?*

---

### **3. From Top-Down to Bottom-Up**

**~1 min**

Đây là **problem statement + research gap**.

Visual:

```text
TOP-DOWN

Culture labels
     ↓
Model
     ↓
"Does it recognize culture?"

vs.

BOTTOM-UP

Physical / biological / chemical facts
     ↓
Representation
     ↓
"Does culture emerge?"
```

Sau đó introduce RQ:

> **Can culinary cultural boundaries emerge from non-semantic natural-science features?**

Đây là slide cực quan trọng.

**Đừng vội nói 304D.**

---

### **4. Our Answer: A Non-Semantic Culinary Representation**

**~45s**

Bây giờ mới reveal:

> **304 dimensions**
>
> Geography + Entity + Odor

Một câu:

> *Instead of encoding what a recipe means, we encode what physically characterizes it.*

Sau đó show 304D.

**Transition:**

> *So how do we turn a recipe into such a representation?*

---

### **5. From Recipe to 304D Vector**

**~1 min**

Đây là pipeline tổng.

```text
50,510 Recipes
       ↓
Ingredient Canonicalization
       ↓
 ┌─────────┬─────────┬─────────┐
 ↓         ↓         ↓
Geography Entity    Odor
248D       28D       28D
 └─────────┴─────────┴─────────┘
              ↓
           304D
              ↓
         K-means
              ↓
     Cultural Alignment
```

Ở đây mention:

* Zenodo dataset
* LLM canonicalization
* GBIF / USDA / FooDB

**Không giải thích công thức ở slide này.**

---

### **6. Three Physical Signals**

**~1:30**

Đây là **methodology detail duy nhất cần đào sâu**.

Ba cột:

#### 🌍 Geography — 248D

> *Where can the biological ingredients exist?*

Intuition của formula:

> Each country gets a score based on how compatible its biological distribution is with the ingredients of the dish.

#### 🥕 Entity — 28D

> *What kinds of food entities compose the dish?*

→ proportion of USDA food categories.

#### 👃 Odor — 28D

> *What chemical aroma profile can emerge?*

→ ingredients + cooking reactions → aroma descriptors → entropy.

Cuối slide:

> **No text embedding.
> No cultural label is used to construct the vector.**

Câu này tui muốn **highlight cực mạnh**.

---

### **7. How Do We Know It Contains Culture?**

**~45s**

Đây là Evaluation setup.

Đừng gọi là “Metrics”.

Hỏi audience:

> **If we never tell the model what culture is, how do we know it recovered anything cultural?**

Rồi:

**External:**

> Do clusters align with geographic labels?

→ PA / Soft-Macro F1

**Internal:**

> Are clusters actually coherent?

→ Silhouette / DB / CH

**Ablation:**

> Where does the signal come from?

→ Geo / Entity / Odor / Fusion

---

# Và từ đây bài bắt đầu “pay off”

### **8. Cultural Boundaries Emerge — Especially at Fine Granularity**

**~1:20**

Đây là **main result slide**.

Show **Full Fusion vs FoodBERT**.

Narrative:

> At continent level, FoodBERT can still perform well.

> But as we move from continents → regions → countries, the picture changes.

Animation/concept:

```text
K = 6
Continents
     ↓
K = 31
Sub-regions
     ↓
K = 207
Countries
```

Rồi **country-level result thật to**:

> **~2× Soft-Macro F1**

và:

> **Higher Partial-Match Accuracy**

Core interpretation:

> **Semantic representations capture broad geographic concepts well, but physical grounding becomes increasingly useful as cultural boundaries become finer.**

Đây là **hero result của toàn bài**.

---

### **9. Where Does the Signal Come From?**

**~1:10**

Ablation.

Show:

**Geo Only | Entity Only | Odor Only | Full Fusion**

Câu hỏi:

> *Is geography doing all the work?*

Và reveal:

> **No.**

Entity và Odor vẫn chứa cultural signal ở finer granularities.

Đây là lúc nói:

> Culinary identity is not encoded only in geography.
> Structural and chemical properties also carry cultural information.

Sau đó:

> **Unexpectedly, Geo-only sometimes beats Full Fusion.**

Một câu thôi.

→ naïve fusion isn't necessarily optimal.

---

### **10. But Is This Just a Clustering Trick?**

**~50s**

Đây là internal metrics.

Tui **không muốn dành hẳn một slide to đùng cho 3 metrics**.

Chỉ cần framing:

> *External alignment could still be misleading if our clusters are fragmented.*

→ show internal figure.

Highlight:

> **Silhouette up to 0.908**

và:

> proposed representations maintain compact, separated clusters.

Kết luận:

> **The cultural alignment is supported by actual geometric structure in the representation space.**

---

### **11. From Natural Science to Cultural AI**

**~45s**

Đây là synthesis + application + conclusion.

Không cần “Conclusion” kiểu academic.

Headline:

> **Culture can leave measurable traces in the physical world.**

Ba takeaway:

**1.** Culinary culture can be represented without semantic embeddings.

**2.** Ecological, structural and chemical signals can recover fine-grained cultural patterns.

**3.** Such a representation can become a prior for culturally-aware AI.

Rồi application:

> recommendation
> retrieval
> cold-start recipes
> cultural AI

Cuối cùng:

> **Making AI understand culture may require looking beyond language.**

🔥

Đây là câu tui muốn audience nhớ.

---

# Như vậy toàn bài có 4 act

Để bro dễ nhớ:

### **ACT I — WHY**

**Slides 1–3**

> Culture → physical world → why bottom-up?

### **ACT II — HOW**

**Slides 4–6**

> 304D → pipeline → three signals

### **ACT III — DOES IT WORK?**

**Slides 7–10**

> evaluation → main result → ablation → internal validation

### **ACT IV — SO WHAT?**

**Slide 11**

> natural science → cultural AI

---

## Một điểm tui rất muốn bro giữ

**Đừng để title của slide nói “Methodology”, “Experimental Setup”, “Results and Discussion”.**

Đó là structure của **paper**.

Presentation nên dùng những câu hỏi như:

> **Can Nature Explain Culture?**

> **Can Culture Emerge Without Semantics?**

> **How Do We Encode a Recipe Without Language?**

> **Does the Representation Actually Recover Culture?**

> **Where Does the Signal Come From?**

> **Is This More Than a Clustering Trick?**

> **What Does This Mean for Cultural AI?**

Nó sẽ làm bài **có narrative hơn rất nhiều**.

---

### Và nếu agent sắp làm PPT, tui sẽ khóa `01_structure.md` theo đúng 11 slide này.

**Tui không nghĩ bro cần brainstorm thêm structure nữa.** Cái cần làm tiếp là biến mỗi slide thành một specification rất cụ thể: **headline + 1 key message + visual + exact content + speaker transition + timing**. Sau đó script và visual spec sẽ sinh ra rất dễ.

# 낚시성 기사 판별 서비스
기사의 제목과 본문을 입력하여 낚시성 기사의 여부를 확인할 수 있는 서비스  
URL을 입력하거나 기사의 제목과 본문을 직접 입력하여 낚시성 기사 여부를 판별할 수 있다.  
낚시성 기사인 경우, 낚시성 기사인 이유와 제목 관련 기사 3개를 검색하여 제공한다.  
비낚시성 기사인 경우, 기사의 본문을 요약하여 제공한다. 

## 🎯 목표
- **낚시성 기사 판별**: Fine-Tuning된 모델을 사용해 기사 분류  
- **낚시성 이유 생성**: RAG를 활용해 외부 정보를 바탕으로 설명 생성  
- **비낚시성 기사 요약**: 요약 모델을 활용해 간결한 본문 요약 생성  


서비스 흐름도는 아래와 같다.  
![image](https://github.com/user-attachments/assets/daf28828-3a9d-4270-b5c7-62934dad56d2)



## 📌 주요 기능

-  뉴스 제목 및 본문에 대한 낚시성 여부 판별  
-  AI 모델 기반 기사 분류 (Clickbait / Not Clickbait)  
-  결과 시각화 및 설명 제공  
- 🛠 RESTful API 제공 (추후 웹 UI 연동 가능)  


## ⚙️ 기술 스택

| 구분 | 기술 |
|------|------|
| Backend | Python, FastAPI |
| AI/ML | Hugging Face Transformers (BERT / LLaMA 등), PyTorch |
| Data | AIHub Clickbait Detection Dataset (JSON) |


## 🧾 데이터셋
사용 데이터: AI Hub의 낚시성 기사 탐지 데이터  
newsTitle, newsContent, clickbaitClass만 추출하여 사용  
낚시성 기사인 경우 - clickbaitClass:0  
비낚시성 기사인 경우 - clickbaitClass:1


예시
```
{
    "newsTitle": "연합뉴스 포털 노출중단 위기에 노조 \"최악의 참사\"",
    "newsContent": "포털 뉴스제휴평가위원회제휴평가위가 연합뉴스에 한 달 포털 노출중단 제재 및 재평가퇴출평가에 해당하는 벌점을 의결하자 전국언론노조 연합뉴스 지부연합뉴스 노조가 입장을 냈다 (생략)",
    "clickbaitClass": 0
}
```

## 🔬 학습 및 추론 방식 (BERT vs LLaMA)

- Llama 3.1 8B model을 finetunning하여 낚시성 기사 여부 판별에 활용  
- 분류가 아닌 생성의 형태를 활용하여 낚시성 기사인지 판별  
- LoRA, Bits and Bytes 4bit 양자화, TRL의 SFTtrainer를 활용하여 학습을 최적화  
- colab A100 환경에서 학습 진행

### ✅ BERT  
- `title + content`를 tokenizer로 인코딩 후 입력  
- `[CLS]` 토큰의 출력 벡터를 기반으로 softmax → 이진 분류  
- 출력 예시: `[0.12, 0.88]` → label = 1 (Clickbait)

### ✅ LLaMA  
- Prompt 형태 입력:  
  ```
  Classify: 제목 본문  
  Answer:
  ```
- 다음 토큰으로 `"Clickbait"` 또는 `"Not Clickbait"` 생성  
- 생성된 텍스트를 파싱하여 결과 판단  


  
## 📊 모델 성능 비교  
분류 모델 대비 생성 모델이 어느 정도의 성능을 가졌는지 비교

결과는 아래와 같다.  
| 모델 | Accuracy | F1 Score | Precision | Recall |
|------|----------|----------|-----------|--------|
| BERT (분류 전용) | 75.81% | 75.49% | 75.80% | 75.81% |
| LLaMA (생성형 모델) | 76.98% | **79.31%** | **81.75%** | **77.02%** |

> ✅ **LLaMA 모델은 F1, Precision, Recall 모두 BERT보다 높음.**  
> 특히 Precision 81.75%는 낚시성 판단에 있어 높은 신뢰도를 의미  
> 실제 성능 수치에서도 F1 Score 79.31%로 BERT를 초과


이후 학습량을 조절하며 추가적인 성능 개선 진행

## ❓ 왜 LLaMA를 선택했는가?

1. **긴 기사 수용 가능**:  
   - BERT는 512토큰 한계  
   - LLaMA는 8,000+ 토큰 수용 → 긴 기사도 분석 가능  

2. **맥락 기반 추론 능력**:  
   - Clickbait 여부는 단순 문장 분석이 아닌 은유, 과장, 문맥 해석 필요  
   - 추론 기반 생성형 모델인 LLaMA가 더 유리  

3. **다기능 확장성**:  
   - 하나의 모델로 다음 작업도 가능  
     - `reason:` → 낚시성 이유 생성  
     - `summarize:` → 본문 요약  
   - 추후 하나의 통합 모델로 inference 가능성 확보 

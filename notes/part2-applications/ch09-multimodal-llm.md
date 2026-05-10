# 第9章 多模态LLM

## 核心概念

> 从"只读文字"到"看图说话"——多模态是 LLM 迈向通用 AI 的关键一步。

### 9.1 视觉 Transformer（ViT）

将 Transformer 架构从文本扩展到图像。

**核心思想**：把图像当作"句子"，把图像块当作"词元"。

```
原始图像 (224×224×3)
  → 切割为 16×16 的小块（196 个 patch）
  → 每个 patch 展平为向量
  → 线性映射 + 位置编码
  → 送入标准 Transformer 编码器
  → 输出：图像的语义表示
```

ViT vs CNN：
| 维度 | CNN | ViT |
|------|-----|-----|
| 感受野 | 局部（卷积核） | 全局（自注意力） |
| 数据需求 | 较少 | 大量预训练数据 |
| 扩展性 | 受限 | 随规模持续提升 |

### 9.2 多模态嵌入模型

将不同模态（文本、图像）映射到**共享向量空间**，使跨模态检索成为可能。

```
CLIP 模型（OpenAI）：

图像编码器 (ViT) ─→ 图像向量 ─┐
                               ├→ 对比学习 → 拉近匹配对，推远不匹配对
文本编码器 (Transformer) → 文本向量 ─┘
```

**CLIP 的训练**：4 亿个 (图像, 文本描述) 对，通过对比学习让匹配对相似度高、不匹配对相似度低。

**CLIP 的能力**：
- 零样本图像分类（不需要训练数据）
- 图文检索（以文搜图、以图搜文）
- 图像语义嵌入

**其他多模态嵌入模型**：
- **ALIGN**（Google）：更大规模噪声数据训练
- **SigLIP**：用 Sigmoid 替代 Softmax，训练更高效

### 9.3 让文本生成模型具备多模态能力

三种主流方案：

| 方案 | 代表模型 | 思路 |
|------|---------|------|
| 交叉注意力融合 | Flamingo | 图像特征通过交叉注意力注入 LLM |
| 投影对齐 | LLaVA | 视觉编码器 → 投影层 → LLM 输入空间 |
| 原生多模态 | GPT-4V、Gemini | 从头训练，统一处理多模态 |

**LLaVA 架构详解**（最经典的方案）：

```
图像 → ViT 编码器 → 视觉特征向量
                         ↓
                    投影层 (MLP)
                         ↓
                    视觉 token（与文本 token 同维度）
                         ↓
              [视觉 token + 文本 token] → LLM → 输出
```

LLaVA 的训练两阶段：
1. **预训练对齐**：冻结 LLM，只训练投影层，用图文对数据
2. **指令微调**：解冻 LLM，用高质量多模态指令数据

**多模态应用场景**：
- 图像描述生成（Image Captioning）
- 视觉问答（VQA）
- 文档理解（OCR + 推理）
- 图表分析
- 代码截图 → 代码

## 代码示例

```python
# CLIP 零样本图像分类
from transformers import CLIPProcessor, CLIPModel
from PIL import Image

model = CLIPModel.from_pretrained("openai/clip-vit-base-patch32")
processor = CLIPProcessor.from_pretrained("openai/clip-vit-base-patch32")

image = Image.open("photo.jpg")
texts = ["一只猫", "一只狗", "一辆汽车", "一朵花"]

inputs = processor(text=texts, images=image, return_tensors="pt", padding=True)
outputs = model(**inputs)

probs = outputs.logits_per_image.softmax(dim=1)
for text, prob in zip(texts, probs[0]):
    print(f"{text}: {prob.item():.2%}")

# 多模态嵌入 - 图文相似度
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("clip-ViT-B-32")
img_emb = model.encode(Image.open("photo.jpg"))
text_emb = model.encode("一只可爱的猫咪")

from sklearn.metrics.pairwise import cosine_similarity
sim = cosine_similarity([img_emb], [text_emb])[0][0]
print(f"图文相似度: {sim:.3f}")

# LLaVA 视觉问答（伪代码）
from transformers import LlavaForConditionalGeneration, AutoProcessor

model = LlavaForConditionalGeneration.from_pretrained("llava-hf/llava-1.5-7b-hf")
processor = AutoProcessor.from_pretrained("llava-hf/llava-1.5-7b-hf")

conversation = [
    {"role": "user", "content": [
        {"type": "image"},
        {"type": "text", "text": "描述这张图片中的内容"},
    ]},
]
prompt = processor.apply_chat_template(conversation, add_generation_prompt=True)
inputs = processor(images=image, text=prompt, return_tensors="pt")
output = model.generate(**inputs, max_new_tokens=200)
print(processor.decode(output[0], skip_special_tokens=True))
```

## 个人思考

1. 多模态模型的"看图"能力到底有多强？图像理解的瓶颈在哪？
2. CLIP 在中文场景下的效果如何？需要中文 CLIP 模型吗？
3. 多模态 RAG：结合第8章，用图像和文本一起做检索和生成
4. 视频理解是下一个前沿——如何处理时序信息？

# 06: 双塔微调 + ONNX int8 导出 + 组件向量重算

**What to build:** 在云 GPU 上对 bge-small-zh 做 LoRA 微调（仅微调、不从头训练），把微调后的 query 编码器合并导出为 ONNX int8（≤30MB），并用它重算 2,708 个组件的描述文档向量，产出浏览器端可加载的模型与 `vectors.bin`。

**Blocked by:** 01 (数据抽取器与特征词典), 05 (合成训练数据生成器)

**Status:** ready-for-agent

- [ ] 用合成训练集在云 GPU（按量或 Kaggle/Colab）完成 bge-small-zh LoRA 微调
- [ ] 合并 LoRA → 导出 ONNX → int8 量化，模型 ≤30MB
- [ ] 用微调后模型重算组件向量，输出 `vectors.bin`（2,708 行 × 512 维，行序与 `components.json` 一致）
- [ ] 产物版本化（模型版本 + 向量版本 + 生成时间），可追溯、可回滚
- [ ] 训练/评测脚本与主工程分离，不进入网站前端与服务器运行时

## Comments

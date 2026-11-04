# OCR Smoke: Local (CPU, M1 Metal off/ARM editable install) vs Qianfan Remoto

Data: 2026-11-04
Hardware local: macOS (Metal não usado pelos pipelines PaddleX nesta corrida), weights em `/Volumes/AILabs/Models/paddlex/2026/official_models` (`PADDLE_PDX_CACHE_HOME`).
Arquivos: 3 PDFs reais (faturas comerciais Ecuador customs), 3 páginas cada = 9 páginas/pipeline, DPI 200.

```
/Volumes/AILabs/Datasets/Docs/MX/Invoices/customs/ecuador/TIM-16542-FACTURA COMERCIAL-8998-00023034.pdf
/Volumes/AILabs/Datasets/Docs/MX/Invoices/customs/ecuador/TIM-19305-FACTURA.COMERCIAL-FY2601064.pdf
/Volumes/AILabs/Datasets/Docs/MX/Invoices/customs/ecuador/TIM-20919-FACTURA.COMERCIAL-28681-DLR.pdf
```

## Resultado (n=9 páginas cada)

| Pipeline | Execução | Média/pág | Mediana | Min | Max | Soma 9 pág |
|---|---|---|---|---|---|---|
| PP-OCRv6 | Local | 42.26s | 42.31s | 19.67s | 61.06s | 380.33s |
| PP-StructureV3 | Local | 153.35s | 137.41s | 86.58s | 228.12s | 1380.13s |
| **PP-StructureV3** | **Qianfan remoto** | **28.63s** | 24.1s | 12.8s | 53.5s | — |
| PaddleOCR-VL-1.6 | Local | 411.38s | 265.36s | 151.21s | 1185.52s | 3702.46s |
| PaddleOCR-VL-0.9b | Qianfan remoto | 33.17s | 30.3s | 14.8s | 69.3s | — |

Taxa ok=9/9 em todos os 5 pipelines (zero falhas).

## Conclusão

Remoto (Qianfan) ganha em velocidade em todos os pares comparáveis:
- PP-StructureV3: remoto 5.4x mais rápido que local (28.6s vs 153.4s/pág).
- PaddleOCR-VL: remoto 12.4x mais rápido (33.2s vs 411.4s/pág) — **mas não é apples-to-apples**: local testado foi VL-1.6 (maior/mais recente), remoto só expõe VL-0.9b (menor). Qianfan não hospeda bare `paddleocr-vl`, `paddleocr-vl-1.5/1.6`, `pp-ocrv6`, nem `pp-ocrv5` bare — todos retornam `401 invalid_model` (confirmado via chamada real). Só `pp-structurev3` e `paddleocr-vl-0.9b` funcionam remoto.

PaddleOCR-VL (qualquer versão) é a pior opção local — VLM pesado, CPU sem GPU/Metal, max chegou a 1185s numa página.

Local só compensa quando:
- precisa de modelo que não existe remoto (PP-OCRv6, PP-StructureV3 local sem custo por chamada, VL-1.6);
- não quer dependência de rede/custo por request.

## Precisão/adequação por tipo de documento (análise, não benchmarked aqui)

- **PP-OCRv6**: texto puro, rápido, sem layout/tabela. Bom pra docs simples de 1 coluna.
- **PP-StructureV3**: layout+tabela+fórmula+OCR. Melhor custo/benefício pra faturas/documentos estruturados (tabelas de itens, cabeçalhos). Recomendado para faturas comerciais como as testadas.
- **PaddleOCR-VL**: modelo VLM, mais "entendimento" semântico, mas caro computacionalmente local. Único caso de uso que justifica é documento muito irregular/manuscrito onde layout fixo falha — não é o caso de faturas padronizadas.

## Bugs corrigidos durante este smoke (ver commits)

1. Qianfan payload faltava campo `"model"` — causava 400 em qualquer uso de `--ppocr_source qianfan` com PP-StructureV3/PaddleOCR-VL. Corrigido em 3 arquivos do `mcp_server/paddleocr_mcp/inference/`.
2. `outputImages` do Qianfan são URLs assinadas (bcebos), não base64 — smoke script salvava garbage. Corrigido + 62 imagens já geradas re-baixadas sem custo extra de API (URLs ainda válidas).

## Saídas

- Local: `output/smoke-2026-11-04/{PP-OCRv6,PP-StructureV3,PaddleOCR-VL-1.6}/<doc>/p<N>/`
- Remoto: `output/smoke-2026-11-04-qianfan/{pp-structurev3,paddleocr-vl-0.9b}/<doc>/p<N>/` (document.md, document.html, output.json, imagens)

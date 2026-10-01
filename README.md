# Hello World com TensorFlow Lite Micro no ESP32

Exemplo mínimo de inferência de machine learning em microcontrolador usando o **TensorFlow Lite Micro (TFLM)** sobre o **ESP-IDF**, simulado no **Wokwi**. Um modelo pequeno, quantizado em int8, aproxima a função seno: a cada 500 ms o firmware calcula `x` entre 0 e 2π, executa a inferência e imprime `y ≈ sen(x)` no monitor serial.

## Estrutura do projeto

| Arquivo | Função |
|---|---|
| `main.cc` | Ponto de entrada (`app_main`): chama `setup()` uma vez e `loop()` a cada 500 ms |
| `main_functions.cc / .h` | Carrega o modelo, registra operadores, aloca tensores e executa a inferência |
| `constants.cc / .h` | Intervalo de x (`kXrange` = 2π) e número de inferências por ciclo (`kInferencesPerCycle` = 20) |
| `output_handler.cc / .h` | Saída dos resultados via `MicroPrintf` (pode ser trocada por LED, display etc.) |
| `model.cc / .h` | Modelo `.tflite` convertido em vetor C (`g_model`, 2488 bytes) |
| `CMakeLists.txt` | Registro dos fontes no componente ESP-IDF |
| `idf_component.yml` | Dependência `espressif/esp-tflite-micro` |

## Requisitos

- ESP-IDF instalado e ativado (`export.sh` / `export.ps1`)
- Conta no [Wokwi](https://wokwi.com) (web) ou extensão Wokwi no VS Code
- Acesso à internet na primeira compilação (para baixar o componente `esp-tflite-micro`)

## Como reproduzir

1. Criar o projeto a partir do exemplo:
   ```bash
   idf.py create-project-from-example "espressif/esp-tflite-micro:hello_world"
   cd hello_world
   ```
2. Definir o chip-alvo (conforme a placa usada no Wokwi):
   ```bash
   idf.py set-target esp32      # ou esp32s3
   ```
3. Compilar:
   ```bash
   idf.py build
   ```
4. Criar os arquivos do Wokwi (`wokwi.toml` e `diagram.json`) apontando para o firmware gerado em `build/` (`.bin` e `.elf`), ou abrir a pasta com a extensão Wokwi no VS Code.
5. Iniciar a simulação e abrir o monitor serial.

## Saída esperada

Linhas no formato:

```
x_value: <x>, y_value: <y>
```

- `x` varia de 0 a 2π em 20 passos (≈ 0,314 por passo)
- `y` acompanha aproximadamente `sen(x)`, com pequeno erro de quantização
- Um ciclo completo dura cerca de 10 s (20 inferências × 500 ms)

> Inserir aqui o print da tela do Wokwi:
> `![Hello World no Wokwi](docs/wokwi.png)`

## Como funciona

1. **`setup()`**: mapeia o modelo (`GetModel`), registra apenas o operador `FullyConnected`, cria o `MicroInterpreter` com uma tensor arena estática e chama `AllocateTensors()`.
2. **`loop()`**: calcula `x` a partir de `inference_count`, quantiza para int8 usando `scale` e `zero_point` do tensor de entrada, executa `Invoke()`, desquantiza a saída e chama `HandleOutput(x, y)`.

## Ajustes comuns

| O que mudar | Onde |
|---|---|
| Duração do ciclo | `kInferencesPerCycle` em `constants.cc` e o `vTaskDelay` em `main.cc` |
| Tamanho da memória do modelo | `kTensorArenaSize` em `main_functions.cc` |
| Forma de saída | `HandleOutput` em `output_handler.cc` |

## Pontos de atenção

- **Tensor arena:** `kTensorArenaSize = 2000` é justa. Se aparecer `AllocateTensors() failed`, aumente para 2048–4096.
- **Tratamento de erros:** se `setup()` falhar, `input` e `output` ficam nulos e `loop()` pode travar o chip. Vale logar o erro e verificar nulidade no `loop()`.
- **Quantização:** a conversão de x para int8 não arredonda nem limita ao intervalo [-128, 127]. Aceitável nesta faixa de valores.
- **Versão da dependência:** `idf_component.yml` usa `version: '*'`. Fixe a versão testada para garantir reprodutibilidade.
- **Comentário no `CMakeLists.txt`:** cita o projeto `micro_speech` por herança do exemplo original.

## Licença e créditos

Código baseado nos exemplos do TensorFlow Lite Micro, © The TensorFlow Authors, sob licença Apache 2.0.

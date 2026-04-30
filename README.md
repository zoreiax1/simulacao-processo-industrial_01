# Projeto de Controle Avançado: Tanque de Aquecimento com Tubulação Longa

Este repositório contém a modelagem, simulação e sintonia de malhas de controle para um processo termodinâmico composto por um tanque de mistura com aquecimento acoplado a uma tubulação longa de descarga. O projeto foi desenvolvido em Python utilizando o ecossistema científico (SciPy, NumPy, Control) e Plotly para renderização de análises dinâmicas.

## 1. Arquitetura do Sistema e Desafio de Controle

O sistema físico apresenta uma dicotomia clássica de controle de processos:
* **Tanque ($T_1$):** Dinâmica rápida, baseada em inércia de mistura, com atraso de transporte praticamente nulo ($\theta \approx 0$). Responde instantaneamente à ação da válvula de aquecimento ($z$).
* **Extremidade da Tubulação ($T_f$):** Ponto de consumo final. Sofre de um atraso de transporte dominante (tempo morto massivo de $\approx 50$ metros de tubulação).

O desafio consiste em garantir que a temperatura da água entregue no ponto de consumo ($T_f$) rastreie o *Setpoint* exigido e rejeite distúrbios de carga (como flutuações na temperatura externa $T_{inf}$), contornando as severas restrições de estabilidade impostas pelo tempo morto.

## 2. Metodologia de Identificação (Modelagem FOPTD)

Para obter a inércia dinâmica do sistema e permitir o projeto analítico dos controladores, aplicou-se um degrau em malha aberta. Os parâmetros do modelo de Primeira Ordem com Atraso de Transporte (FOPTD) foram extraídos utilizando o **Método de Smith (Regra dos 28,3% e 63,2%)**. 

Este método foi escolhido pela sua superioridade imunológica a ruídos assintóticos no regime permanente, utilizando a interpolação normalizada na fase de maior inclinação da curva de resposta:
$$\tau = 1.5 \cdot (t_{63} - t_{28})$$
$$\theta = t_{63} - t_{degrau} - \tau$$

**Parâmetros Físicos Extraídos:**
* **Tanque ($T_1$):** $K_p = 0.7619$ °C/%, $\tau = 0.0954$ min, $\theta = 0.0000$ min.
* **Tubulação ($T_f$):** $K_p = 0.6901$ °C/%, $\tau = 0.1174$ min, $\theta = 0.4896$ min.

## 3. Estratégia de Controle e Sintonia (SIMC / Vapt-Vupt)

Os PIDs foram projetados com base na teoria de **Internal Model Control (IMC)**, especificamente utilizando as regras de Skogestad (SIMC).

### 3.1. Malhas SISO (Single-Input Single-Output)

Testes de stress individuais (rastreamento de alvo e rejeição a distúrbios) foram conduzidos para provar a limitação da arquitetura SISO neste contexto:

* **Controle Direto do Tanque ($T_1$):**
    * **Estratégia:** Resposta Suave ($\tau_c = \tau$).
    * **Sintonia:** $K_c = 1.312$ | $T_i = 0.0954$ min.
    * **Desempenho:** Altamente agressivo e estável contra distúrbios de base, mas ignorante à perda térmica ao longo da tubulação.
* **Controle Direto da Tubulação ($T_f$):**
    * **Estratégia:** Limitado por Atraso Dominante ($\tau_c = \theta$).
    * **Sintonia:** $K_c = 0.173$ | $T_i = 0.1174$ min.
    * **Desempenho:** Extremamente letárgico. A necessidade de achatar o ganho ($K_c$) para evitar ciclos de instabilidade destrói a capacidade do sistema de rejeitar distúrbios térmicos rapidamente.

### 3.2. Solução Definitiva: Controle em Cascata 2-DOF

Para fundir a agressividade regulatória do Tanque com a garantia de *Setpoint* da Tubulação, implementou-se uma **Malha em Cascata**. 

A sintonia do Controlador Mestre não pode ser feita utilizando a planta original, pois ele enxerga uma "Planta Equivalente" alterada pelo fechamento da malha Escrava. Adotou-se o método sequencial baseado no **COBEM 2003 e Skogestad Half-Rule**:
* $K_{eq} = 0.9058$
* $\tau_{eq} = 0.1651$ min
* $\theta_{eq} = 0.5373$ min

Adicionalmente, para evitar *overshoot* provocado pelo atraso durante a navegação do Setpoint, o Controlador Mestre foi implementado com **2 Graus de Liberdade (2-DOF - Metodologia Orlando Arrieta)**, introduzindo o fator de ponderação $b$ na ação proporcional:
$$P = K_c \cdot (b \cdot SP - PV)$$

**Sintonia Final da Cascata:**
* **Escravo ($T_1$):** $K_{c2} = 1.312$, $T_{i2} = 0.0954$, $b_2 = 1.0$ (Rastreamento imediato).
* **Mestre ($T_f$):** $K_{c1} = 0.170$, $T_{i1} = 0.1651$, $b_1 = 0.4$ (Agressivo contra distúrbios, suave na mudança de alvo).

O resultado visual comprova a aniquilação de desvios térmicos (modo regulatório) operando no limite físico da saturação do aquecedor, sem oscilações estruturais.

## 4. Requisitos e Execução (Reprodução do Ambiente)

O projeto requer isolamento de dependências. O arquivo `requirements.txt` assegura a exatidão das bibliotecas necessárias, incluindo o motor de renderização de gráficos embutidos no Jupyter.

**Instruções via Terminal (PowerShell):**

1.  Clone o repositório:
    ```powershell
    git clone [https://github.com/SEU_USUARIO/SEU_REPOSITORIO.git](https://github.com/SEU_USUARIO/SEU_REPOSITORIO.git)
    cd SEU_REPOSITORIO
    ```
2.  Crie e ative o ambiente virtual:
    ```powershell
    python -m venv .venv
    .\.venv\Scripts\Activate.ps1
    ```
3.  Instale as dependências:
    ```powershell
    pip install -r requirements.txt
    ```
4.  Abra o arquivo `Tanque_tubulacao.ipynb` no VSCode e certifique-se de selecionar o kernel do `(.venv)` no canto superior direito para permitir a execução da biblioteca Plotly via `ipykernel`.
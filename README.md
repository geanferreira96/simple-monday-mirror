# Simple Monday — Revisão de Funcionalidades

O **Simple Monday** é um cliente premium para gerenciamento de tempos e marcações de ponto corporativo, unificando e sincronizando dados de três ecossistemas distintos em tempo de execução:
1. **Monday.com** (tanto via API pública quanto via credenciais de API interna).
2. **TOTVS Carol Clock In** (batidas físicas por geolocalização e relógio nativo).
3. **TOTVS Meu RH** (espelho de ponto oficial, banco de horas e justificativas).

Abaixo está o detalhamento técnico de cada módulo e suas respectivas funcionalidades.

---

## 1. Calendário e Timeline Diária (`calendar_page_v2.py`)
A tela de Calendário é o núcleo operacional do usuário, exibindo a distribuição diária de horas e fornecendo os caminhos para apontamentos e envio de ponto.

*   **Timeline Horária Dinâmica**:
    *   **Algoritmo de Distribuição em Cascata**: Detecta sobreposições reais de horários e distribui as caixas de tempo em colunas (`col_idx`), deslocando-as lateralmente em incrementos de `24px`. Isso garante que pelo menos uma aba lateral de cada cartão fique visível e clicável, mesmo que vários horários colidam.
    *   **Empilhamento por Duração**: Ordena a renderização no Flet `Stack` pela duração em ordem decrescente, garantindo que marcações mais curtas fiquem fisicamente sobrepostas no topo de marcações mais longas.
    *   **Opacidade de Vidro (Glassmorphism)**: Ajusta a opacidade de colunas secundárias para `0.70` (semitransparente) para facilitar a visualização de apontamentos posicionados por baixo.
*   **Alternador de Visualização (Timeline vs. Lista)**:
    *   Um botão segmentado nativo do Cupertino permite alternar a exibição para o modo **Lista**.
    *   Nesta exibição, cada cartão representa um período do dia e recebe **bordas e sombras reativas com base no status de colisão** (Verde para períodos válidos e Vermelho para sobreposições de horários). Clicar no cartão abre os detalhes ou formulário de edição correspondente.
*   **Apontamento Manual no Monday**:
    *   Organizado em um layout proporcional de 60/40 com campos flexíveis e dropdowns esticados por completo.
    *   **Esforço Cupertino Alternável**: Permite ao usuário escolher a duração de uma tarefa deslizando as rodas do `CupertinoTimerPicker` ou digitando os valores diretamente no teclado numérico via `TextField` com reatividade total nos cálculos.
    *   **Ações "Dividir e Add" e "Add Após"**: O botão "Add Após" cria um segmento subsequente cujo horário de início herda automaticamente o horário de término da tarefa anterior, garantindo consistência cronológica em um clique.
    *   **Validador de Colisão**: Executa a checagem em tempo real de conflitos horários entre os segmentos do diálogo e entre as tarefas já lançadas na timeline do dia, destacando caixas inválidas em rosa-avermelhado suave.
*   **Envio de Ponto em Lote (Meu RH)**:
    *   Permite lançar múltiplos cartões de batida (data, hora e sentido: Entrada/Saída) na mesma janela, enviando-os sequencialmente sob uma justificativa unificada.
    *   **Sugestões Inteligentes**: Apresenta chips de sugestões cronológicas baseados nos seus turnos configurados, ocultando sugestões que coincidam ou estejam em um raio de 10 minutos de marcações ativas.
    *   **Trava de Segurança do Ciclo**: Bloqueia preventivamente a abertura do diálogo e limita a data mínima do seletor nativo `CupertinoDatePicker` ao primeiro dia do ciclo ativo (ex: dia 16 do mês atual se o ciclo fecha no dia 15).

---

## 2. Dashboard e Métricas de Ponto (`dashboard_page.py` & `dashboard_service.py`)
Módulo analítico que calcula a saúde das suas horas no período selecionado.

*   **Consolidação de Ponto Misto**: Unifica batidas obtidas do **Carol Clock In** e do **Meu RH** usando mesclagem inteligente para calcular as horas extras, atrasos e saldos acumulados de forma simétrica com a timeline do Calendário.
*   **Cálculo Automatizado de Horas Extras a 100%**: Identifica finais de semana e feriados nacionais, computando integralmente as batidas realizadas nestes dias como **hora extra (100%)** sem descontar meta diária.
*   **Gráfico de Evolução e Meta Dinâmica**: Renderiza um gráfico de barras responsivo com uma linha pontilhada de meta que se ajusta automaticamente ao valor configurado em suas preferências (ex: meta de 8 horas, 6 horas).
*   **Botões Didáticos de Informação**: Mini-ícones de informação posicionados no canto inferior de cada cartão de métrica, abrindo alertas informativos com a regra de negócio aplicada a cada somatório.

---

## 3. Painel de Perfil Cupertino Reativo (`profile_page_v2.py`)
Centraliza os dados cadastrais do usuário integrados aos portais remotos.

*   **Chaveador de Perfil Cupertino**: Se as contas do **Monday** e do **Meu RH** estiverem conectadas ao mesmo tempo, um seletor Cupertino iOS nativo é renderizado no topo para navegar entre as contas.
*   **Perfil Enriquecido do Meu RH**: Tenta carregar e parsear dados da API TOTVS RM de forma multicamadas e resiliente. Exibe informações ricas de vínculo e pessoais:
    *   *Matrícula, Cargo, Departamento, Gestor Direto, Apelido, Cidade Natal, Gênero e Data de Nascimento.*
*   **Persistência e TTL**: Os dados de perfil são persistidos no banco local SQLite em cache, respeitando as preferências de TTL (tempo de vida) em minutos para evitar chamadas de rede desnecessárias a cada inicialização de tela.

---

## 4. Importação Instantânea por QR Code (`main.py`, `shell.py`, `encryption_service.py`)
Permite migrar credenciais entre dispositivos móveis e desktop com segurança absoluta.

*   **Criptografia Forte AES-256**: Um serviço implementado em Python puro criptografa o conjunto de tokens do Monday e credenciais do Clock In e Meu RH em um payload serializado de alta segurança.
*   **QR Code e Deep Link Receptor**: Gera um código visual usando a URI personalizada `simplemonday://connect?data=<payload>`. A rota do aplicativo intercepta a inicialização a partir do Deep Link, decifra os dados de forma transparente e valida as integrações em um instante.
*   **Scanner Integrado com Câmera Nativa**: Disponibiliza um scanner de câmera nativo (exclusivo para Android, iOS e Web) com decodificação rápida via `pyzbar` para ler o QR Code diretamente pela lente física do dispositivo.

---

## 5. Painel de Configurações Geral (`settings_page.py`)
Mapeia e persiste as regras e parâmetros operacionais do aplicativo.

*   **Ponto**:
    *   *Dia de Início do Ciclo*: Seletor deslizante de 1 a 31 determinando o fechamento do período do ponto.
    *   *Meta de Horas Diária*: Regulagem móvel da jornada de trabalho ideal por dia de semana.
    *   *Grade de Turnos*: Permite adicionar, remover e ajustar turnos com seletores Cupertino independentes para Entrada/Saída, indicando a duração e validando inconsistências horárias (entrada posterior à saída).
    *   *Sugestões de Justificativa*: Permite adicionar, editar, remover e ordenar os presets de justificativas que serão exibidos na folha de bater ponto do Meu RH.
    *   *Alerta de Batida Ímpar*: Switch para ligar/desligar alertas visuais no calendário sobre pendências.
*   **Notificações**:
    *   *Notificações de Turno*: Liga notificações locais dinâmicas de saída que calculam em tempo real quando você deve bater o ponto com base nos registros feitos hoje.
    *   *Lembretes Fixos*: Switch que solicita permissões locais de notificação no seu celular e gerencia disparos de alarmes em horários fixos customizados nos dias de trabalho.
*   **Apontamentos**:
    *   *Jornada Diária*: Permite definir limites de duração mínima e máxima por dia.
    *   *Origem do Intervalo*: Escolha segmentada entre basear o assistente de sugestões de apontamento na timeline a partir de suas "Batidas de Ponto reais" ou com base nos seus "Turnos fixos configurados".
*   **Atualizações**:
    *   Configuração do tempo de cache (TTL em minutos) antes do aplicativo invalidar os dados locais e buscar atualizações nos servidores remotos do Monday, Carol Clock In e Meu RH.

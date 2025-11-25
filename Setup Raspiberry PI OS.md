### Tutorial: Instalação Manual do OctoPrint no CB1 (Placa Não-Raspberry Pi)

O CB1 (Control Board One) da Big Tree Tech é um substituto para o Raspberry Pi (especificamente o Compute Module 4). Como o **kernel do CB1 é diferente** do kernel do Raspberry Pi, **não é possível** utilizar a imagem de cartão SD pré-configurada do OctoPi. Consequentemente, a instalação do OctoPrint no CB1 deve ser realizada manualmente via SSH, um processo que é **muito mais trabalhoso** do que o uso de uma imagem pré-fabricada.

Este tutorial descreve o processo de instalação manual do OctoPrint no CB1, usando a imagem mínima da BTT e as instruções manuais do fórum OctoPrint.

#### Etapa 1: Preparação e Instalação da Imagem Mínima do SO (Fonte: GitHub BTT CB1)

A primeira etapa é obter o sistema operacional correto para o CB1 no repositório GitHub da Big Tree Tech (BTT):

1.  **Obtenção da Imagem Mínima:** Você deve buscar a **versão mínima** da imagem de sistema, que é baseada no sistema operacional Debian Linux.
    *   Exemplos de imagens mínimas de kernel disponíveis incluem: `CB1_Debian11_bullseye_minimal_kernel5.16_20230215.img.xz` e `CB1_Debian11_bullseye_minimal_kernel5.16_20220929.img.xz`, que adicionam apenas o *shell script* para configuração de Wi-Fi via cartão SD, baseadas no Debian puro.
    *   Para o CB1, a BTT mantém sistemas Kernel 5.16. Imagens com o Clipper pré-instalado (como Klipper, Moonraker, Mainsail e KlipperScreen) também estão disponíveis, mas a instalação manual do OctoPrint exige o uso da **imagem mínima**.
2.  **Flash da Imagem:** Use o **Raspberry Pi Imager** oficial. Selecione a imagem mínima baixada (escolhendo *custom image* ou imagem personalizada do seu computador) e grave-a no cartão micro SD (o tamanho recomendado é de 16 GB). Ignore quaisquer mensagens solicitando a formatação do cartão após o processo.
3.  **Configuração Wi-Fi:** O cartão SD terá uma partição acessível. Edite o arquivo **`system.cfg`** e insira o nome da sua rede Wi-Fi e a senha.
    *   O *shell script* de Wi-Fi é otimizado no CB1 para corrigir problemas, como Wi-Fi com criptografia híbrida WAP2/WAP3 ou SSIDs contendo espaços (por exemplo, `WIFI_SSID="CB1 Tester"`).
4.  **Boot Inicial:** Insira o CB1 no adaptador Pi 4B e o cartão SD no adaptador. Ligue-o.
5.  **Acesso Terminal/SSH:** Após o *boot*, você pode acessar o terminal ou usar o SSH (via Putty).
    *   Os detalhes de login padrão são: **Nome de usuário: biqu** e **Senha: biqu**.
	*   A versão mais novo da imagem mudou o processo de definição de senha. Agora, após a instalação e primeiro acesso é necessário definir uma senha nova.
    *   Se estiver usando uma versão antiga do SO, pode ser necessário desativar o acesso SSH para o usuário `root` (o acesso SSH é desativado por padrão para o usuário `root` nas versões mais recentes do SO BTT).

#### Etapa 2: Acesso via SSH e Instalação Manual do OctoPrint

Como você não pode usar a imagem OctoPi, você deve seguir a seção **"installing manually"** (instalação manual) nas instruções do OctoPrint. O guia de instalação manual da comunidade OctoPrint pressupõe que o usuário tenha um **entendimento mais do que básico da linha de comando Linux**.

1.  **Processo de Instalação:** A instalação envolve **copiar e colar os comandos, um de cada vez,** no terminal Putty (SSH) e executá-los.
2.  **Instalação de Dependências:** O guia manual exige a instalação de dependências como Python 3.9 a 3.13 e `pip`, e a criação de um ambiente virtual (`venv`) para evitar conflitos de dependência. Comandos típicos incluem:
    ```bash
    cd ~
    sudo apt update
    sudo apt install python3 python3-pip python3-dev python3-setuptools python3-venv git libyaml-dev build-essential libffi-dev libssl-dev
    mkdir OctoPrint && cd OctoPrint
    python3 -m venv venv
    source venv/bin/activate
    pip install --upgrade pip wheel
    pip install octoprint
    ```
3.  **Configuração de Usuário Crítica (CB1):** Ao configurar os usuários, você deve **mudar o nome de usuário de `pi` para `biqu`**. O usuário `pi` não existe no CB1.
    *   O guia padrão do OctoPrint usa o usuário `pi` para conceder acesso às portas seriais (tty e dialout), mas no CB1 você deve adaptar isso para o usuário `biqu`.
4.  **Início do Servidor:** Após a instalação, você pode iniciar o servidor OctoPrint usando o comando `octoprint serve` dentro do ambiente virtual.
5.  **Resultado:** Após concluir as etapas, reinicie o sistema.

Agradeço o feedback! É fundamental incluir as etapas opcionais, porém cruciais, para uma experiência completa e segura com o OctoPrint, especialmente a configuração da câmera e o uso de um *reverse proxy*.

A seguir, apresento a **revisão e expansão da `#### Etapa 3: Configuração e Uso Final`** do seu tutorial de instalação manual do OctoPrint no CB1, incorporando os detalhes sobre *Autostart*, *HAProxy*, *Webcam* e *Comandos de Sistema*, conforme detalhado nas fontes.

#### Etapa 3: Configuração e Uso Final (Expandido)

Após a conclusão da instalação manual do OctoPrint via SSH, é necessário configurar o acesso, o início automático e, opcionalmente, a *webcam* e o *proxy reverso* para otimizar o uso do CB1.

1.  **Acesso à Interface Web:**
    *   Insira o endereço IP no seu navegador (ex: `http://<IP do CB1>:5000`) para acessar o assistente de configuração (*setup wizard*) do OctoPrint.
    *   Embora possa haver algumas etapas de configuração extras em comparação com a imagem pré-fabricada do OctoPi, o resultado é uma **versão funcional do OctoPrint**.

2.  **Configuração de Início Automático (*Autostart*):**
    Para que o OctoPrint inicie automaticamente no *boot* (o que é recomendado, já que o processo manual de instalação não o configura por padrão), crie o arquivo `/etc/systemd/system/octoprint.service`.

    *   **Adaptação CB1:** No arquivo de serviço, o parâmetro `User` deve ser definido como **`BQ`** (o nome de usuário padrão do CB1) em vez de `pi`.
    *   Após criar o arquivo, habilite o serviço usando `sudo systemctl enable octoprint.service`.

3.  **Configuração de Usuário para Portas Seriais e Comandos:**
    Para garantir que o usuário **`BQ`** tenha acesso às portas seriais (necessário para comunicação com a placa da impressora), e para que o OctoPrint possa executar comandos de reinicialização e desligamento, siga estas etapas:

    *   **Portas Seriais:** Embora não explicitamente detalhado para o CB1, o guia de instalação manual indica que o usuário deve ser adicionado aos grupos `tty` e `dialout` para acessar as portas seriais.
        *   **Adaptação CB1:** Adapte os comandos de grupo substituindo `pi` por `BQ`.
    *   **Comandos de Sistema:** No UI do OctoPrint, em `Settings > Commands`, configure os comandos para reiniciar o OctoPrint (`sudo service octoprint restart`), reiniciar o sistema (`sudo shutdown -r now`) e desligar o sistema (`sudo shutdown -h now`).
    *   **Sudoers:** Se for necessário rodar comandos sem senha, o guia sugere adicionar regras *sudoers* para permitir que o usuário **`BQ`** utilize `shutdown` e `service` sem exigir senha.

4.  **Acessibilidade na Porta 80 e Uso de HAProxy (*Reverse Proxy*):**
    É recomendado usar o **HAProxy** como um *reverse proxy* em vez de configurar o OctoPrint para rodar na porta 80. Isso tem diversas vantagens, como: evitar que o OctoPrint precise de privilégios de `root` para se ligar à porta 80 e permitir que o *stream* de vídeo (`mjpg-streamer`) também seja acessível na porta 80.

    *   **Instalação do HAProxy:** Instale o pacote usando `sudo apt install haproxy`.
    *   **Configuração:** O CB1 usa uma imagem baseada em Debian 11 (Bullseye), portanto, a configuração do HAProxy 2.x deve ser utilizada em `/etc/haproxy/haproxy.cfg`.
        *   Esta configuração roteia as requisições para o OctoPrint (`127.0.0.1:5000`) e permite que o *mjpg-streamer* (webcam) seja acessível pelo *path* `/webcam/` no endereço da porta 80.
    *   **Habilitação:** Modifique `/etc/default/haproxy` e defina **`ENABLED`** como **`1`**. Inicie o serviço com `sudo service haproxy start`.
    *   **URL Localhost:** Para segurança adicional, após configurar o HAProxy, você pode configurar o OctoPrint para escutar apenas no *loopback interface* (localhost) adicionando: `server: host: 127.0.0.1` no arquivo `~/.octoprint/config.yaml`.

5.  **Instalação Opcional de Webcam (MJPG-Streamer):**
    Para suporte à *webcam* e *timelapse*, é necessário baixar e compilar o **MJPG-Streamer** manualmente.

    *   **Instalação de Dependências:** Instale as dependências necessárias (que dependem da versão do Debian). A lista de pacotes para Debian atual inclui `subversion`, `libjpeg62-turbo-dev`, `imagemagick`, `ffmpeg`, `libv4l-dev`, `cmake`.
    *   **Compilação:** Clone o repositório (`git clone https://github.com/jacksonliam/mjpg-streamer.git`), navegue até o diretório experimental e execute `make`.
    *   **Configuração no OctoPrint:** Se estiver usando o HAProxy, no OctoPrint (Webcam & Timelapse settings), o **Stream URL** deve ser alterado para uma URL relativa: `/webcam/?action=stream`.
        *   O **Snapshot URL** deve ser mantido como `http://127.0.0.1:8080/?action=snapshot`.
        *   O `Path to FFMPEG` é geralmente `/usr/bin/ffmpeg`.
    *   **Início Automático da Webcam:** Para iniciar a *webcam* automaticamente, crie um *script* `webcamDaemon` e um arquivo de serviço `webcamd.service` em `/etc/systemd/system/`. Novamente, assegure-se de que o campo `User` no arquivo `webcamd.service` seja configurado para **`BQ`**.
    *   **Segurança (Porta 8080):** Como o `mjpg-streamer` não permite o *bind* para o *localhost* apenas, se o OctoPrint estiver acessível pela internet, é crucial bloquear o acesso à porta 8080 de todas as fontes exceto o *localhost* usando **`iptables`**. Use comandos `iptables` e `ip6tables` e torne-os persistentes instalando `iptables-persistent` e salvando as regras.

> **Observação de Desempenho:** O CB1 (com seu processador Cortex A53 e 1 GB de RAM) é comparável em especificações a um Raspberry Pi 3B, o que é **suficientemente bom para rodar o OctoPrint**. No entanto, a necessidade de instalação manual o torna menos conveniente do que o Raspberry Pi, onde uma imagem pré-feita está disponível.

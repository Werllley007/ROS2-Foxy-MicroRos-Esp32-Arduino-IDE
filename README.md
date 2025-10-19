# Tutorial de Configuração: Ambiente de Desenvolvimento Micro-ROS (ROS Foxy/Docker)

Este guia detalha a configuração completa do ambiente de desenvolvimento dentro de um contêiner Docker (`ros-foxy`), incluindo a instalação do Arduino IDE, bibliotecas essenciais e o agente Micro-ROS, focando na integração serial.

---

## 1. Configuração do Arduino IDE

Devido às restrições de execução de AppImage em contêineres Docker, o procedimento envolve a extração manual do executável e a configuração de dependências.

### 1.1. Dependência Essencial

Instale a biblioteca `libfuse2` no seu sistema (Host ou Contêiner):

```bash
sudo apt install libfuse2
```

### 1.2. Transferência do Instalador
Transfira o arquivo AppImage do seu Host para o contêiner Docker:

```bash
docker cp ./arduino-ide_2.3.6_Linux_64bit.AppImage ros-foxy:/home/werlley/
```
### 1.3. Solução Alternativa: Extração Manual e Execução
Execute os comandos abaixo dentro do contêiner ros-foxy:

Extrair o Conteúdo:

```bash
./arduino-ide_2.3.6_Linux_64bit.AppImage --appimage-extract
# Isso criará uma pasta chamada 'squashfs-root'.
Localizar e Executar o Binário:

```bash
# Procura pelo binário principal executável dentro da pasta extraída
find . -type f -name "arduino-ide" -executable

# Exemplo de execução (ajuste o caminho se necessário):
```

```bash
/home/werlley/squashfs-root/arduino-ide --no-sandbox
```

### 1.4. Atalho no .bashrc
Adicione um alias ao seu arquivo ~/.bashrc para simplificar a inicialização do IDE:

```bash
alias arduinoide='/home/werlley/squashfs-root/arduino-ide --no-sandbox'
```
### 1.5. Configuração da Placa ESP32
Referência: https://randomnerdtutorials.com/installing-the-esp32-board-in-arduino-ide-windows-instructions/

Dentro do Arduino IDE, adicione a URL no campo “Additional Board Manager URLs” (Arquivo > Preferências):

URL:
[https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json](https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json)

## 2. Configuração do Pyserial
Instale o pip e a biblioteca pyserial no contêiner para comunicação serial:

```bash
sudo apt update
sudo apt install pip -y
sudo pip install pyserial
```
## 3. Configuração da Biblioteca Micro ROS Arduino
Esta biblioteca é essencial para a integração com o Micro-ROS.

### 3.1. Download e Transferência
Baixe a biblioteca Micro ROS Arduino:
https://github.com/micro-ROS/micro_ros_arduino

Transfira o arquivo ZIP para o contêiner:

```bash
docker cp ./micro_ros_arduino-kilted.zip ros-foxy:/home/werlley/
```
### 3.2. Configuração Dentro do Contêiner (ros-foxy)
Execute os comandos a seguir dentro do contêiner:

Instalar unzip (se necessário):

```bash
sudo apt update && sudo apt install unzip -y
```
Criar o Diretório Padrão (se não existir):

```bash
mkdir -p ~/Arduino/libraries
```
Extrair e Renomear a Pasta:

```bash
cd /home/werlley
unzip micro_ros_arduino-kilted.zip
mv micro_ros_arduino-kilted micro_ros_arduino
```
Mover para o Diretório de Bibliotecas:

```bash
mv /home/werlley/micro_ros_arduino /home/werlley/Arduino/libraries/
```
# A pasta da biblioteca agora está em /home/werlley/Arduino/libraries/

## 4. Configuração do Micro ROS SETUP e Agent
Execute os passos a seguir dentro do contêiner para configurar o workspace ROS 2 e compilar o Agente.

Referência: https://github.com/micro-ROS/micro_ros_setup

### 4.1. Configuração do Workspace

```bash
source /opt/ros/$ROS_DISTRO/setup.bash

mkdir uros_ws && cd uros_ws

git clone -b $ROS_DISTRO [https://github.com/micro-ROS/micro_ros_setup.git](https://github.com/micro-ROS/micro_ros_setup.git) src/micro_ros_setup

rosdep update && rosdep install --from-paths src --ignore-src -y

colcon build

source install/local_setup.bash
```
### 4.2. Compilação do Micro-ROS Agent

```bash
ros2 run micro_ros_setup create_agent_ws.sh
ros2 run micro_ros_setup build_agent.sh
source install/local_setup.sh
```
## 5. Rodar e Testar o Micro-ROS Agent
### 5.1. Rodar o Micro-ROS Agent (Contêiner Separado)

O Agent deve ser executado em um Terminal Host separado, utilizando a imagem microros/micro-ros-agent.

Referência: https://micro.ros.org/docs/tutorials/core/first_application_linux/

Comando Host:

```bash
docker run -it --rm \
  -v /dev:/dev --privileged \
  --net=host \
  microros/micro-ros-agent:foxy serial --dev /dev/ttyACM0 -b 115200
```
# Verifique se a porta é /dev/ttyACM0 ou 8889 (se for UDP).

### 5.2. Testes de Conectividade ROS 2
Enquanto o Agent estiver rodando, acesse seu contêiner ros-foxy (Terminal 2) e utilize os comandos de teste ROS 2:

```bash
# Sem seu usuário
alias edocker='clear && docker exec -it --user werlley -e DISPLAY=$DISPLAY -e QT_X11_NO_MITSHM=1 -e LIBGL_ALWAYS_SOFTWARE=1 ros-foxy bash

# Com seu usuário
alias edocker='clear && docker exec -it --user werlley -e DISPLAY=$DISPLAY -e QT_X11_NO_MITSHM=1 -e LIBGL_ALWAYS_SOFTWARE=1 ros-foxy bash -c "cd /home/werlley && bash"'

# Dentro do conteiner
nano .bashrc

# Cole no bashrc
source ~/uros_ws/install/local_setup.bash 

# Dentro do contêiner:
ros2 topic list
ros2 node list
ros2 topic echo <topic>
ros2 node info <node>
```

## 6. Configuração Final do Contêiner com Acesso Serial
Após concluir as instalações, salve o estado do seu contêiner e recrie-o com acesso permanente à porta serial para o usuário werlley.

### 6.1. Salvar o Estado do Contêiner

```bash
# 1. Parar o contêiner ros-foxy
docker stop ros-foxy

# 2. Salvar o estado atual em uma nova imagem
docker commit ros-foxy ros-foxy-pronto
```

### 6.2. Recriação com Acesso Serial (/dev/ttyACM0 ou /dev/ttyUSB0)
Primeiro, pare e remova o contêiner antigo para liberar o nome:

```bash
docker stop ros-foxy
docker rm ros-foxy
```

Agora, execute o novo contêiner a partir da imagem ros-foxy-pronto, incluindo a flag --device apropriada (execute um dos comandos abaixo no Terminal Host):

**Opção 1**: Porta /dev/ttyACM0

```bash
docker run -it \
  --name ros-foxy \
  --user werlley \
  -e DISPLAY=$DISPLAY \
  -e QT_X11_NO_MITSHM=1 \
  -e LIBGL_ALWAYS_SOFTWARE=1 \
  --device /dev/ttyACM0:/dev/ttyACM0 \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  ros-foxy-pronto bash
```

**Opção 2**: Porta /dev/ttyUSB0

```bash
docker run -it \
  --name ros-foxy \
  --user werlley \
  -e DISPLAY=$DISPLAY \
  -e QT_X11_NO_MITSHM=1 \
  -e LIBGL_ALWAYS_SOFTWARE=1 \
  --device /dev/ttyUSB0:/dev/ttyUSB0 \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  ros-foxy-pronto bash
```

Se o seu dispositivo **/dev/ttyACM0** ou **/dev/ttyUSB0** estiver conectado, este comando deve finalmente iniciar seu ambiente de desenvolvimento completo e pronto para o Micro-ROS!


# Material Didático: Balanceamento de Carga e Proxy Reverso com Nginx

## 1. Introdução e Objetivo
O objetivo deste material é apresentar, na prática, o funcionamento de um balanceador de carga (Load Balancer). Um Load Balancer atua como um "porteiro" para a sua infraestrutura, distribuindo o tráfego de requisições web entre múltiplos servidores (back-ends). Isso garante que nenhuma máquina fique sobrecarregada, aumenta a disponibilidade do sistema e melhora o tempo de resposta para o usuário final.

**Nota sobre a infraestrutura:** Utilizaremos o Docker e o Docker Compose como ferramentas de apoio para subir nossos servidores rapidamente. Pense no Docker aqui como uma "caixa preta" que liga as máquinas virtuais para o nosso teste; os fundamentos avançados de contêineres podem ser estudados separadamente.

---

## 2. A Arquitetura do Laboratório
Vamos simular uma arquitetura clássica de mercado:
* **1 Servidor Nginx** atuando como Load Balancer e Proxy Reverso, recebendo as requisições externas na porta **8080**.
* **3 Servidores Web (Back-ends)** atuando como "trabalhadores" (workers). Eles receberão as requisições repassadas pelo Nginx.

Para facilitar a visualização de qual servidor está respondendo a cada momento, usaremos uma aplicação web simples (`traefik/whoami`) que imprime na tela diversas informações da requisição, incluindo o seu próprio ID (Hostname).

---

## 3. Preparando o Ambiente Base

Para iniciar, crie uma pasta chamada `lab-loadbalance` e crie os dois arquivos abaixo dentro dela:

### 3.1. Arquivo `nginx.conf`
Este é o arquivo de configuração central do Nginx. Ele define o grupo de servidores e como o tráfego será roteado.

```nginx
events {}

http {
    # Define o grupo de servidores back-end
    upstream meus_servidores {
        server backend1:80;
        server backend2:80;
        server backend3:80;
    }

    server {
        # O Nginx vai escutar na porta 8080 (acessível externamente)
        listen 8080;

        location / {
            # Repassa a requisição para o grupo "meus_servidores"
            proxy_pass http://meus_servidores;
        }
    }
}
```

### 3.2. Arquivo `docker-compose.yml`
Esta é a "receita" de infraestrutura que instanciará o Nginx e os três servidores de aplicação.

```yaml
version: '3'

services:
  # Nosso Load Balancer Nginx
  loadbalancer:
    image: nginx:latest
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    ports:
      # Expõe a porta 8080 do host para a 8080 do contêiner
      - "8080:8080"
    depends_on:
      - backend1
      - backend2
      - backend3

  # Servidores Back-end executando a aplicação de teste
  backend1:
    image: traefik/whoami
  backend2:
    image: traefik/whoami
  backend3:
    image: traefik/whoami
```

Para ligar a infraestrutura, abra o terminal na pasta do projeto e execute:
```bash
docker-compose up -d
```

---

## 4. Testando Estratégias de Balanceamento de Carga

O Nginx permite configurar diferentes algoritmos para decidir qual servidor receberá a próxima requisição. Todas as alterações abaixo devem ser feitas no bloco `upstream` do arquivo `nginx.conf`. Após cada alteração, reinicie o Nginx com o comando `docker-compose restart loadbalancer`.

### 4.1. Round Robin (Padrão)
Se nenhum algoritmo for especificado, o Nginx usa o Round Robin (distribuição circular). Ele entrega uma requisição para cada servidor, em fila (Backend 1 -> Backend 2 -> Backend 3 -> Backend 1...).
* **Teste:** Execute `curl http://localhost:8080` várias vezes e observe o campo `Hostname` mudando sequencialmente.

### 4.2. Round Robin com Peso (Weighted)
Usado quando temos servidores com capacidades de hardware diferentes. Podemos enviar mais tráfego para a máquina mais potente.
```nginx
    upstream meus_servidores {
        server backend1:80 weight=3; # Recebe 3 vezes mais tráfego
        server backend2:80;
        server backend3:80;
    }
```

### 4.3. Persistência de Sessão (IP Hash)
Em aplicações que guardam sessão na memória local do servidor (stateful), o usuário precisa cair sempre na mesma máquina. O `ip_hash` calcula um identificador baseado no IP do cliente.
```nginx
    upstream meus_servidores {
        ip_hash;
        server backend1:80;
        server backend2:80;
        server backend3:80;
    }
```
* **Teste:** Ao fazer requisições sucessivas, o `Hostname` retornado será sempre o mesmo.

### 4.4. Least Connections (Menor Número de Conexões)
Envia a nova requisição para o servidor que tiver o menor número de conexões ativas no momento, evitando sobrecarregar uma máquina que está processando requisições demoradas.
```nginx
    upstream meus_servidores {
        least_conn;
        server backend1:80;
        server backend2:80;
        server backend3:80;
    }
```

---

## 5. Segurança e Rastreabilidade

O Nginx não serve apenas para distribuir carga; ele protege os servidores e enriquece as requisições.

### 5.1. Proteção contra Sobrecarga (Rate Limiting)
Podemos mitigar ataques de negação de serviço (DoS) limitando quantas requisições um IP pode fazer por segundo.

*Configuração:*
```nginx
http {
    # Cria a zona de controle de requisições limitando a 1 por segundo
    limit_req_zone $binary_remote_addr zone=limitacao:10m rate=1r/s;

    upstream meus_servidores {
        server backend1:80;
        server backend2:80;
        server backend3:80;
    }

    server {
        listen 8080;
        location / {
            # Aplica a regra com uma pequena margem (burst)
            limit_req zone=limitacao burst=2 nodelay;
            proxy_pass http://meus_servidores;
        }
    }
}
```
* **Teste:** Execute um laço de repetição rápido: `for i in {1..10}; do curl -i http://localhost:8080; done`. As primeiras passarão (status 200), e as próximas serão bloqueadas (status 503).

### 5.2. Preservação da Identidade do Cliente (Headers de Proxy)
Como o tráfego passa pelo Nginx, o servidor back-end acha que todas as requisições vêm do IP do Nginx. Para que a aplicação conheça o IP real do usuário (para auditoria e logs), o Nginx deve injetar cabeçalhos HTTP.

*Configuração:*
```nginx
    server {
        listen 8080;
        location / {
            proxy_pass http://meus_servidores;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
```
* **Teste:** Execute um `curl http://localhost:8080` e verifique na saída da aplicação os campos `X-Real-Ip` e `X-Forwarded-For` preenchidos.

---

## 6. Limitações da Versão Open Source

Durante estudos de sistemas distribuídos, é comum falar sobre algoritmos baseados em métricas dinâmicas. O Nginx suporta recursos avançados, porém, é preciso compreender as limitações do modelo de negócios da ferramenta:

* **Least Response Time (Menor Tempo de Resposta):** O Nginx possui a diretiva `least_time`, que roteia requisições para o servidor mais rápido. Porém, essa função é **exclusiva do Nginx Plus** (versão comercial paga). Na versão gratuita, o `least_conn` atua como a melhor alternativa disponível.
* **Algoritmos Adaptativos (Adaptive Routing):** O Nginx Open Source não possui roteamento adaptativo nativo que analise uso de CPU ou memória do back-end. 

**Como o mercado contorna isso?**
Empresas que não adquirem o Nginx Plus frequentemente utilizam extensões como o **OpenResty** (que integra a linguagem Lua ao Nginx para criar regras dinâmicas personalizadas) ou delegam esse tipo de roteamento inteligente para ferramentas de **Service Mesh** (como Istio ou Linkerd) em ambientes Kubernetes.

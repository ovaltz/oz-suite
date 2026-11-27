# OZ Keys Mobile: conexão por IP com o PC

O OZ Keys Mobile (Android) fala com o PC por duas conexões diretas, sempre
pela rede local (Wi-Fi/cabo do mesmo roteador). Não passa por servidor na
internet.

| O quê | Programa no PC | Porta | Protocolo | Precisa parear? |
|---|---|---|---|---|
| Botões (Cenas) | OZ Keys | 47591 | HTTP | Sim (PIN) |
| Chat e alertas | OZ Live | 22112 | WebSocket | Não (só leitura) |

O endereço é sempre `IP-do-PC:porta`, por exemplo `192.168.1.10:47591`.
O app aceita colar com `http://` ou `ws://` na frente e completa a porta
padrão quando ela não vem.

## 1. Botões: celular → OZ Keys (porta 47591)

O OZ Keys abre um servidor HTTP em `0.0.0.0:47591` assim que liga.

**Pareamento (uma vez por aparelho)**
1. No PC: OZ Keys → Celular → mostra o QR Code e um **PIN de 6 dígitos**
   (vale 5 minutos e só pode ser usado uma vez).
2. O celular lê o QR (que traz o IP e a porta) ou a pessoa digita o IP.
3. O celular manda `POST /api/parear` com o PIN e o nome do aparelho.
4. O PC devolve um **token de 32 caracteres**, que fica guardado no
   celular. A partir daí todo pedido leva o cabeçalho `x-oz-token`.

**Rotas usadas pelo app**

| Rota | Para quê |
|---|---|
| `GET /api/ping` | Confere se o PC está no ar (sem token) |
| `POST /api/parear` | Troca o PIN pelo token (sem token) |
| `GET /api/acoes` | Lista as Cenas na ordem do celular |
| `POST /api/executar/:id` | Aperta um botão (roda a Cena) |
| `POST /api/acoes` · `PUT /api/acoes/:id` · `DELETE /api/acoes/:id` | Cria, edita e apaga Cena |
| `PUT /api/ordem` | Salva a ordem e os botões escondidos |
| `GET /api/perfis` · `POST /api/perfis/:id` | Lista e troca o perfil de atalhos |
| `GET /api/obs/cenas` | Cenas do OBS (para montar um passo pelo celular) |

Os aparelhos pareados aparecem no PC e podem ser removidos de lá. Remover
invalida o token na hora.

## 2. Chat: celular → OZ Live (porta 22112)

O chat **não** passa pelo OZ Keys do PC. O celular conecta direto no
mesmo WebSocket que o OZ Live já usa para o overlay e o dock do OBS:

```
ws://IP-do-PC:22112/
```

- É só leitura, por isso não tem token, o mesmo modelo de confiança do
  `overlay.html` na rede local.
- Antes de salvar, o app faz um `GET http://IP-do-PC:22112/` (o OZ Live
  serve o `dock.html` ali) para responder rápido se deu certo.
- A conexão manda ping a cada 20 s e reconecta sozinha 3 s depois de cair.
- Também aceita o endereço do OZ Live Studio (`ozlive://IP:22125`) colado
  direto.

**Formato de cada evento (JSON)**

```json
{
  "plataforma": "twitch",
  "usuario": "fulano",
  "tipo_de_evento": "chat_mensagem",
  "conteudo": { "texto": "salve!" }
}
```

| `tipo_de_evento` | O que o celular mostra |
|---|---|
| `chat_mensagem` | A mensagem (`conteudo.texto`) |
| `presente` | "mandou 3x Rosa" (`conteudo.quantidade`, `conteudo.presente`) |
| `social` | seguiu / compartilhou / se inscreveu (`conteudo.subtipo`) |
| `novo_membro` | "virou membro do canal" |
| `banner` | Aviso pronto, sem autor (raid, sistema) |

Qualquer outro tipo é ignorado.

**Sem o PC:** se o campo do PC ficar vazio, o app pode ler o chat direto
das plataformas (Twitch, Kick e YouTube pelo nome do canal). Nesse modo
não vêm TikTok nem os alertas, que só o OZ Live do PC junta.

## Se o celular não achar o PC

1. Celular e PC na **mesma rede**. Wi-Fi de visitante e "isolamento de
   clientes" no roteador bloqueiam a conexão.
2. Abra no navegador do celular `http://IP-do-PC:47591/api/ping`. Tem de
   responder `{"app":"okeyz","status":"ok"}`. Para o chat, use
   `http://IP-do-PC:22112/`.
3. Firewall do Windows: a rede tem de estar como **Privada**, e o OZ Keys
   e o OZ Live liberados (o Windows pergunta na primeira vez que abrem).
4. O IP do PC pode mudar quando o roteador reinicia. Se mudou, leia o QR
   de novo. Fixar o IP do PC no roteador evita isso.

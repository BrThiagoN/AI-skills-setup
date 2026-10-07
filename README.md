# AI Skills Setup

Coleção pessoal de skills em pastas simples com `SKILL.md`, mantida no Git para ficar disponível em outros dispositivos e projetos de IA.

## Skills incluídas

| Skill | Uso |
| --- | --- |
| [`megabrain-design`](skills/megabrain-design/) | Direção visual e criação de interfaces com identidade própria. |
| [`senior-backend-dev-workflow`](skills/senior-backend-dev-workflow/) | Fluxo cuidadoso para projetar, implementar e revisar mudanças de backend. |

As pastas de cada skill incluem os arquivos necessários. A skill de backend também usa os documentos em `references/`.

## Usar em outro dispositivo

Clone ou atualize este repositório e copie a pasta completa da skill desejada para o diretório de skills reconhecido pelo harness que você estiver usando. Mantenha `SKILL.md` e os arquivos auxiliares juntos.

O conteúdo segue o formato portátil de skills baseado em `SKILL.md` e não exige dependências próprias. O diretório de instalação e a descoberta automática variam entre harnesses; consulte a configuração da ferramenta escolhida.

## Adicionar uma skill

Crie `skills/<nome-da-skill>/SKILL.md`. Coloque referências, exemplos e outros arquivos usados pela skill dentro da mesma pasta, usando caminhos relativos. Assim o pacote continua fácil de copiar entre dispositivos e ferramentas.

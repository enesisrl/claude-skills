# Jira Worker Plugin

Plugin Claude Code che fornisce il command `/jira-worker` per lavorare automaticamente i ticket di uno spazio Jira.

## Cosa fa

Per ogni ticket selezionato:

1. lo sposta in stato **In Corso**;
2. lo implementa nel codice del progetto corrente con Claude Code;
3. aggiunge un **commento** al ticket con il riepilogo della lavorazione (file toccati, scelte fatte, note per il test);
4. se tutto va a buon fine lo sposta in stato **Testing**; in caso di problemi lo lascia In Corso con un commento esplicativo.

Al termine presenta un riepilogo complessivo. **Non esegue mai commit**: le modifiche restano nel working tree.

## Utilizzo

```
/jira-worker <spazio-jira> [note aggiuntive]
```

- `<spazio-jira>`: project key (es. `PROJ`) o nome del progetto Jira. Se omesso, viene chiesto.
- `[note aggiuntive]` (opzionali, chieste interattivamente se non fornite):
  - **Filtro ticket**: etichette, priorità, assegnatario o JQL custom per selezionare solo certi ticket;
  - **Vincoli tecnici**: istruzioni implementative valide per tutti i ticket (convenzioni, librerie, parti da non toccare).

Prima di iniziare mostra la lista dei ticket trovati e chiede quali lavorare; da lì in poi procede in completa autonomia.

## Requisiti

- Connettore MCP **Atlassian** attivo nella sessione (claude.ai Atlassian), con permessi su Jira.
- Workflow Jira con stati raggiungibili equivalenti a "In Corso" e "Testing" (il command cerca le transizioni per nome in modo tollerante).

# Suivi Gaulois MCP

A remote MCP server that lets an assistant read a volleyball season's statistics. It is read-only: it lists players, matches and numbers, and it cannot change any of them. I built it so I can ask questions about the season in plain language, instead of building a screen for each question.

It is the companion to [Suivi Gaulois](https://github.com/OTTOPro/suivi-gaulois): the app records, this server exposes, the assistant answers.

The code is private, and so is the player data. This page is the public record: what the server does, how it is built, and what is not there. Every name and number below is invented.

## A question, and what happens

> Which players improved their reception over the last three matches?

There is no screen for this, and there does not need to be one. The assistant calls `historique_matchs` to find the last three completed matches and `liste_joueurs` to get the roster, then `fiche_joueur` for each player. A player sheet carries a match by match log, with the reception average of each match and the number of receptions behind it. The assistant compares the three matches and answers:

> Two players improved. Alex Fictif went from 1.4 to 2.1, over 34 receptions. Bruno Exemple went from 1.8 to 2.0, over 22. Charles Test went from 2.3 to 2.6, but on 7 receptions, which is too few to conclude.

The numbers come from the server. The remark about seven receptions is the assistant's, and it can make it because every average is returned with its volume.

## The five tools

| Tool | Returns |
|---|---|
| `liste_joueurs` | The players: id, name, number, position, team, active or not, and how many matches have statistics. |
| `fiche_joueur` | One player: identity, season totals and averages, the match by match log with rating and comment, and a comparison with the team and with the position. |
| `historique_matchs` | The matches in date order: opponent, home or away, status, set scores, result. |
| `resume_match` | One match: set scores, match notes, team totals, and each player's line. |
| `stats_equipe` | One team: win and loss record, team averages, and the three highest and three lowest players for each indicator. |

All five are declared read-only to the client. Creation dates, modification dates and the author of an entry are left out. Everything else is returned.

## How it works

```
assistant
   |
   |  1. POST /mcp without a token
   |     <- 401, with the address of the authorization metadata
   |
   |  2. /authorize, with the assistant's published client identity and PKCE (S256)
   v
+------------------------------- Worker -------------------------------+
|                                                                      |
|  consent page  ->  GitHub sign-in  ->  one allowed account           |
|                                             |                        |
|  3. authorization code  ->  token           v                        |
|     grants and tokens in KV, stored by hash                          |
|                                                                      |
|  4. POST /mcp with the token                                         |
|     tool -> read-only wrapper (one SELECT) -> calculation module     |
|                         |                                            |
+-------------------------|--------------------------------------------+
                          v
                 D1 database of the app
                 (the state document)
```

One Cloudflare Worker plays both OAuth roles: it is the authorization server for the assistant, and the protected resource at `/mcp`. MCP runs over Streamable HTTP, stateless. TypeScript, the Workers OAuth provider library, KV for the grants, Vitest.

**Read-only is enforced in code.** The server reads the same database the coaches write to during a match. Every query goes through one wrapper that accepts a single `SELECT` statement and refuses the rest: any other verb, several statements, comments. The server contains one query, the one that reads the state document. A tool cannot write because there is no path to write through. As far as I know D1 has no read-only binding, so the guarantee lives in that one small file, with its own tests.

**One account.** I am the only user. The server signs in with GitHub and grants a token to a single account, checked by login and by numeric id, because a login can be renamed and then taken by someone else. The check runs again on every call, so changing the allowed account cuts existing tokens off at once. The GitHub token is used once, to read which account signed in, and is not kept.

**Approving is not signing in.** The assistant identifies itself with a published client identity, and the server also accepts dynamic registration. Either way the client introduces itself, so a click on Allow proves only that someone clicked. That click never returns an authorization code. It sends the browser to GitHub, and the code is issued only when GitHub comes back with the allowed account. PKCE with S256 is required on every authorization.

**The local shortcut stays local.** To test without GitHub, a local instance can sign in as a fixed account. The switch is a variable that exists only in the local development file. That file is never deployed, and the variable is absent from the production configuration. A second lock requires the configured public address to be a loopback address. Tests cover each case: without the variable, the authorization page goes to GitHub; with the variable on a deployed address, it still goes to GitHub.

**The numbers are the app's numbers.** The server imports the calculation module of the app. It does not carry its own formulas. Reception average, attack efficiency and serve efficiency are computed by the code that draws the player page. The consequence is a coupling: the server is built from the app's source at deploy time. When the app changes a formula or the shape of its state document, this server has to be rebuilt and redeployed, or the two will disagree. I accept that: a number that differs between the app and the assistant is the failure I want to avoid.

## What is not there

- No writing. Not a training plan, not a note, not a correction.
- One account. No roles, no second coach.
- No cache. Every call reads the state document again.
- No date range. The numbers are for the season to date.
- Counters per match, not per set. The server reads the state document and ignores the live event log.
- One thing not tried in production: signing in with a second GitHub account to see it refused. Tests cover that case on a local instance. The rest runs in production: I connected an assistant to the deployed server, signed in with GitHub, and used the five tools on the real season.

## About the data

The comments and match notes written by the coach go out to the model when it is asked. The instance is private and tied to a single account. The players are students, and exposing their data to a third party service is a decision to make knowingly, not a technical detail.

This page contains no real name, number, result or opponent.

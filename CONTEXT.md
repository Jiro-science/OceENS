# OcéEns

EPF's course evaluation platform: surveys are created per programme, students answer them, and the answers are exported, visualised and summarised.

## Documentation language

The documentation (this file, `README.md`, `docs/`) is written in English. The application itself is French: its interface, and the product vocabulary users see in it, stay in French. When the documentation names one of those product terms, it gives the French word in italics next to the English one the code uses, for example a survey (*sondage*) or a summary (*synthèse*). Interface labels (*Tester*, *Changer d'utilisateur*) and messages the application displays or logs are quoted verbatim, in French, because that is what the reader sees on screen. Nothing else in the documentation is in French.

## Language

### Authentication

**Development login** (*connexion de développement*, `AUTH_MODE=dev`):
Login without an identity provider: you pick a user's e-mail address and are logged in as that user, with no proof of identity. It only exists when `AUTH_MODE=dev` and must never be used in production.
_Avoid_: impersonation, spoofing, fake login

# Git Usage
## Git environment
## Git usage
### Init
**git init**
### Clone code
**git clone ssh/https**
Before we can clone the ssh, we need to config the git environment.
- git config user.name=xxxx
- git config user.email=xxxx

In case, company's proxy server:
- git config http.proxy=http://username:passwd@proxy ip:port
- git conifg https.proxy=https://username:passwd@proxy ip:port

### Push Code
- **git add**
- **git commit -m**
- **git push**   *Need to verify your email before we can git push.*

### Revert
- **git log**
- **git reset --soft xxxx**
- **git push origin master --force**
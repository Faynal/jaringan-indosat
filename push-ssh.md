Github >_Termux
Push file ke Github SSH
```
git init
git add .
git commit -m "Commit pertama dari Termux"
git branch -M main
git remote set-url origin git@github.com:username/proyek-saya.git
git push -u origin main

```

Apabila terjadi error gunakan perintah ini:
```
git push -u origin main --force
```
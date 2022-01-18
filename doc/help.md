# Q&A
## シェル実行のみを行うtemplateを作りたい

1. command と arg を使ってみる
   - 単数行しかできないが見栄えが良い
```yaml
  - name: whalesay
    container:
      image: docker/whalesay:latest
      command: [sh, -c]
      args: ["cowsay hello | tee /mnt/vol/hello_world.txt"]
```

2. command に直接書く
    - 複数行できるのでオススメ
```yaml
  - name: print-hoge
    container:
      image: alpine:latest
      command:
      - sh
      - "-c"
      - |
        echo "hoge"
```

注意点:
- "-c" オプションを忘れるとログが出てこないことがある
- source は使いにくい


---
<details>
<summary>template</summary>

---

</details>

---
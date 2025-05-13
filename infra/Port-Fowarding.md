

```bash
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j REDIRECT --to-port 5000
```
- dport 80 : 80번 포트로 들어오면
- --to-port 5000 : 5000번 포트로 차렷

f
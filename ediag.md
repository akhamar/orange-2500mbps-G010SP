# Port 1

```bash
ediag.exe -b10eng

device 1
nvm cfg
7
35=70
36=70
56=6
59=6
save
exit
```
this sets up 2.5g on port 1


# Port 2

```bash
ediag.exe -b10eng

device 2
nvm cfg
7
35=70
36=70
56=0 or 7?
59=0 or 7?
save
exit
```
this sets up 10g on port 2

# Revert back

```bash
ediag.exe -b10eng

device 1
nvm cfg
7
35=50
36=50
56=7
59=7
save
exit
```

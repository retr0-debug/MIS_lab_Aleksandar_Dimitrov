# Процедура: SSH до сървъра само с личен ключ, без вход като root (DST-05)

| Поле | Стойност |
|---|---|
| Реализира контрол | DST-05, SSH частта: PasswordAuthentication no, PermitRootLogin no |
| Annex A | 8.5, 8.2, 5.17, 8.9 |
| R-ID | R-11 (изтекли администраторски пароли, A-13, A-15) |
| Изпълнява се от | HQ администратор |
| Честота | При първоначална настройка и при всяка нова машина (A-04, A-05). Проверка (стъпка 9 и отрицателният тест): до 5-о число всеки месец, от Ръководител ИТ. |
| Версия, дата | 0.1, 05.11.2026 |

## Предусловия

| Какво | Стойност |
|---|---|
| Достъп | Акаунт kali на srv (192.168.10.10) с право на sudo; потребител на gw (192.168.10.1), от който се влиза в srv. |
| Инструменти | srv: openssh-server, услугата ssh работи (sudo systemctl enable --now ssh). gw: openssh-client (ssh, ssh-keygen, ssh-copy-id, scp). |
| Снапшот преди започване | pre-h06 на srv (VirtualBox: Machine → Take Snapshot или VBoxManage snapshot srv take pre-h06). |
| Втора сесия | Отваря се в стъпка 3 (ssh kali@192.168.10.10 от gw) и остава отворена до края. Конзолата на srv във VirtualBox е резервен достъп. |

## Стъпки

**1. srv: запиши състоянието преди**

```bash
{ date -Is; hostname; sudo sshd -T | grep -Ei '^(permitrootlogin|passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication) '; } > predi.txt; cat predi.txt
```

Очакван резултат: Дата, srv и четирите текущи стойности (по подразбиране в Kali: permitrootlogin without-password, passwordauthentication yes).

**2. gw: създай личен ключ и го качи на srv**

```bash
test -f ~/.ssh/id_ed25519 || ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519
ssh-copy-id -i ~/.ssh/id_ed25519.pub kali@192.168.10.10
```

Очакван резултат: „Number of key(s) added: 1“ или „All keys were skipped because they already exist“.

**3. gw, нов терминал: отвори втората сесия**

```bash
ssh kali@192.168.10.10
```

Очакван резултат: Промпт на srv. Тази сесия не се затваря до края.

**4. gw: провери, че ключът работи, преди да забраниш паролите**

```bash
ssh -o PasswordAuthentication=no -o BatchMode=yes kali@192.168.10.10 true; echo $?
```

Очакван резултат: 0, без питане за парола. Друго: спри (виж „При отклонение“).

**5. srv: провери, че нищо не надделява над новия файл**

```bash
grep -nEv '^[[:space:]]*(#|$)' /etc/ssh/sshd_config | head -1
sudo grep -rEi '^(permitrootlogin|passwordauthentication|kbdinteractiveauthentication)' /etc/ssh/sshd_config.d/
```

Очакван резултат: Първият ред е „Include /etc/ssh/sshd_config.d/*.conf“. Втората команда не връща нищо.

**6. srv: създай настройката в отделен файл**

```bash
printf 'PermitRootLogin no\nPasswordAuthentication no\nKbdInteractiveAuthentication no\n' | sudo tee /etc/ssh/sshd_config.d/10-dst05.conf
```

Очакван резултат: Отпечатват се трите реда; файлът /etc/ssh/sshd_config.d/10-dst05.conf съществува.

**7. srv: провери синтаксиса**

```bash
sudo sshd -t; echo $?
```

Очакван резултат: Няма съобщение, после 0.

**8. srv: приложи без да прекъсваш сесиите**

```bash
sudo systemctl reload ssh
# във втората сесия:
echo alive
```

Очакван резултат: Няма грешка; втората сесия отпечатва alive.

**9. srv: запиши състоянието след (от работещата услуга)**

```bash
{ date -Is; hostname; sudo sshd -T | grep -Ei '^(permitrootlogin|passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication) '; } > sled.txt; cat sled.txt
```

Очакван резултат: permitrootlogin no, passwordauthentication no, kbdinteractiveauthentication no, pubkeyauthentication yes.

## Проверка

**Положителна (разрешеното работи):**

```bash
ssh -o BatchMode=yes kali@192.168.10.10 hostname
```

От gw, нова връзка. Очаквано: srv, без питане за парола.

**Отрицателна (забраненото се отказва):** от gw, нова връзка (не от отворената сесия)

```bash
{ date -Is; ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password kali@192.168.10.10 true; ssh -o BatchMode=yes root@192.168.10.10 true; } 2>&1 | tee otkaz.txt
```

Очаквано: два реда „kali@192.168.10.10: Permission denied (publickey).“ и „root@192.168.10.10: Permission denied (publickey).“, без питане за парола.

След това на srv:

```bash
sudo journalctl -u ssh --since "10 min ago" --no-pager > zhurnal.txt
```

Очаквано: „Accepted publickey for kali“ от положителния тест и отказите от 192.168.10.1 (за root: ROOT LOGIN REFUSED).

## При отклонение

- Стъпка 4 не връща 0: спри, не създавай файла. Повтори стъпка 2 веднъж. Ако пак не е 0, ескалирай към Ръководител ИТ в същия работен ден.
- Стъпка 5 показва друг първи ред или втората команда връща редове: спри, не създавай файла. Ескалирай към Ръководител ИТ в същия работен ден; кой файл се променя решава той.
- Стъпка 7 връща грешка: изпълни sudo rm /etc/ssh/sshd_config.d/10-dst05.conf, не прави reload, ескалирай към Ръководител ИТ в същия работен ден.
- Стъпка 8: втората сесия е прекъснала или положителната проверка не минава: веднага изпълни „Възстановяване“ и ескалирай към Ръководител ИТ до 1 час.
- Стъпка 9 или отрицателният тест не дават очакваното (вход с парола или като root минава): не връщай назад, контролът просто не е наложен. Запиши изхода, ескалирай към Ръководител ИТ в същия работен ден.

## Възстановяване (Rollback)

От втората сесия на srv:

```bash
sudo rm /etc/ssh/sshd_config.d/10-dst05.conf && sudo sshd -t && sudo systemctl reload ssh
sudo sshd -T | grep -Ei '^(permitrootlogin|passwordauthentication) '   # трябва да съвпада с predi.txt
```

Без втора сесия: вход в srv от конзолата на VirtualBox (не минава през sshd) и същите команди. Ако и това е невъзможно: върни снапшота pre-h06.

## Запис

Остават четири файла: predi.txt, sled.txt, zhurnal.txt (на srv) и otkaz.txt (на gw).

```bash
# на gw
scp kali@192.168.10.10:{predi.txt,sled.txt,zhurnal.txt} .
# в хранилището
git add artefakti/s06-soa-procedura && git commit -m "h06: soa i procedura" && git push
```

Пазят се в git най-малко 12 месеца (срокът от DST-05). Изпълнението се записва в тикет: дата, изпълнител, резултат на отрицателния тест.

## Проверка от съученик

| Поле | Стойност |
|---|---|
| Изпълнил, дата | |
| Засякъл на стъпка | |
| Неясно | |
| Отрицателният тест даде | |
| Поправено след проверката | |

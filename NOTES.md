## Notes

### Notes on Firewall with UFW

I was curious about the firewall. Specifically, what would happen if I block access to port 8000 where Parsoid is listening to. Let's see.

```
# Allow SSH
sudo ufw allow ssh

# Allow HTTP
sudo ufw allow http

# Enable UFW
sudo ufw enable

# Check status
sudo ufw status
```

Nope. Does not work. I received this error when I was trying to edit the main page.

```
Error loading data from server: apierror-visualeditor-docserver-http-error: (curl error: 28) Timeout was reached. Would you like to retry?
```

Adding `8000` fixed it.
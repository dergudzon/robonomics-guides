robonomics ipfs-pubsub docker-compose instance

1. Rename `.env_example` to `.env`:
```mv .env_example .env```

2. Launch the docker compose once for generating ipfs default config:
```docker compose up```
When you'll see the `Error: experimental pubsub feature not enabled` line, you can just stop the docker compose by pressing Ctrl+C

3. After this, the folders "ipfs_data" and "ipfs_staging" will appear in the current directory. Open the ipfs config file in the text editor:
```nano ./ipfs_data/config```
 
Find the "Pubsub" option here and turn it on:
```
  "Pubsub": {
    "Enabled": true,
    "DisableSigning": false,
    "Router": "gossipsub"
  },
```

4. That's it, you can just launch your docker project as daemon:
```docker compose up -d```

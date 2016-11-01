SSH DEPLOY

#1 Setup SSH encryption key
  So let’s recap what we need to do to securely prepare our SSH tunnel between our remote host and Travis:
  #1 Generate a dedicated SSH key (it is easier to isolate and to revoke);
    ssh-keygen -t rsa -b 4096 -C '<repo>@travis-ci.org' -f ./deploy_rsa      
  #2 Encrypt the private key to make it readable only by Travis CI (so as we can commit safely too!);
    travis encrypt-file deploy_rsa --add
    @NOTE: for installing travis
      sudo npm install ruby-dev
      sudo gem install travis
  #3 Copy the public key onto the remote SSH host;
    ssh-copy-id -i deploy_rsa.pub <ssh-user>@<deploy-host>
  #4 Cleanup after unnecessary files;
    rm -f deploy_rsa deploy_rsa.pub
  #5 Stage the modified files into Git.
    git add deploy_rsa.enc .travis.yml

#2 Setup SSH decryption in Travis job
  We still need to setup a few things before we are able to enact anything:

  #1 Preventing the SSH interactive prompt when connecting to a host for the first time (which means, for each build);
    addons:
      ssh_known_hosts: <deploy-host>
  #2 Decrypting the encrypted SSH private key; #3 Adding the key to the current ssh-agent to make any SSH-based command agnostic to the private key location.
    before_deploy:
    - openssl aes-256-cbc aes-256-cbc -K $encrypted_<...>_key -iv $encrypted_<...>_iv -in deploy_rsa.enc -out /tmp/deploy_rsa -d            #_
    - eval "$(ssh-agent -s)"
    - chmod 600 /tmp/deploy_rsa
    - ssh-add /tmp/deploy_rsa

  @Notice: we do extract the deploy_rsa in Travis /tmp folder to avoid deploying the decrypted key by any mean.
  Use before_deploy to setup the SSH agent for two reasons:
    - it happens after the script stage: we then avoid any possibility for third party scripts to leak active keys (npm worm, remember?);
    - the build will fail if the deploy setup is incorrect.

#3 Travis deploy script
    deploy:
      provider: script
      skip_cleanup: true
      script:
        - scp -r file_or_folder_path <account>@<hostname>:<path_to_deploy_folder>

#4 Run script on remote host server
  after_deploy:  
    ssh <account>@<hostname> "path_to_script.sh"

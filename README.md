# LibKlustersShared

Library needed to install Klusters and NDManager software on Ubuntu 20.04

## Compile guide

### Dependencies

Install the following dependencies before installing  the library:
```
sudo apt install qt5-default libqt5webkit5-dev build-essential cmake
```

  1. Go to `/LibKlustersShared` directory and type:
  ```
  mkdir build
  cd build
  cmake ..
  make
  sudo make install
  ```

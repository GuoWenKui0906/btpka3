
# install

```sh
# 会创建 /usr/local/rvm
curl -sSL https://get.rvm.io | sudo bash -s stable

# 将使用rvm的用户添加到rvm用户组中
usermod -a -G rvm zll
usermod -a -G rvm git
usermod -a -G rvm root

# 查询已知的可安装组件的版本
rvm list known

# 安装ruby
sudo -i rvm install ruby-2.1.1


# 删除rvm
sudo rm -fr ~/.rvm
sudo rm -fr ~/.rvmrc
sduo rm -fr /usr/local/rvm
sduo rm -fr /etc/profile.d/rvm.sh

```
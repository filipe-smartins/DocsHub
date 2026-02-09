
INSTALL OH MY ZSH

sudo apt install zsh curl git -y

sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"


INSTALL COMAND HISTORY SEARCH (Ctrl + R)
sudo apt install fzf

INSTALL AUTOSUGGESTIONS
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions

INSTALL SYNTAX HIGHLIGHTING
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting

ADD THE PLUGINS TO ZSH
sudo vim ~/.zshrc

plugins=(git fzf zsh-autosuggestions zsh-syntax-highlighting)

RESTART ZSH SERVICE
source ~/.zshrc

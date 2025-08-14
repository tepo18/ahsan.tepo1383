#!/bin/bash

CYAN="\e[36m"
GREEN="\e[32m"
YELLOW="\e[33m"
RESET="\e[0m"

BASE_PATH="$HOME"
DOWNLOAD_DIR="/sdcard/Download/Akbar98"

# تنظیم نام و ایمیل Git یکبار برای همیشه
git config --global user.name "tepo18"
git config --global user.email "tepo18@example.com"

echo -e "${CYAN}Have you already cloned the repo?${RESET}"
echo "1) No, I want to clone a new repo"
echo "2) Yes, repo already cloned"
echo -ne "${CYAN}Enter choice (1 or 2): ${RESET}"
read -r CLONE_CHOICE

if [ "$CLONE_CHOICE" == "1" ] ; then
  echo -ne "${CYAN}Enter the GitHub repo name to clone (e.g. reza-shah1320): ${RESET}"
  read -r REPO_NAME
  REPO_PATH="$BASE_PATH/$REPO_NAME"
  
  if [ -d "$REPO_PATH" ]; then
    echo -e "${YELLOW}Folder '$REPO_PATH' already exists!${RESET}"
  else
    echo -e "${GREEN}Cloning repo from GitHub...${RESET}"
    git clone "https://github.com/tepo18/$REPO_NAME.git" "$REPO_PATH"
    if [ $? -ne 0 ]; then
      echo -e "${YELLOW}Failed to clone repo. Exiting.${RESET}"
      exit 1
    fi
  fi

elif [ "$CLONE_CHOICE" == "2" ] ; then
  echo -ne "${CYAN}Enter the existing cloned repo folder name: ${RESET}"
  read -r REPO_NAME
  REPO_PATH="$BASE_PATH/$REPO_NAME"
  
  if [ ! -d "$REPO_PATH" ]; then
    echo -e "${YELLOW}Folder '$REPO_PATH' not found! Exiting.${RESET}"
    exit 1
  fi
else
  echo -e "${YELLOW}Invalid choice. Exiting.${RESET}"
  exit 1
fi

cd "$REPO_PATH" || { echo -e "${YELLOW}Cannot enter repo folder. Exiting.${RESET}"; exit 1; }

while true; do
  echo -e "\n${CYAN}Select an option:${RESET}"
  echo "1) Create new file"
  echo "2) Edit existing file"
  echo "3) Delete a file"
  echo "4) Commit changes"
  echo "5) Push to GitHub (with sublinks)"
  echo "6) Copy content from download folder to repo file"
  echo "7) Exit"
  echo -ne "${CYAN}Choice: ${RESET}"
  read -r CHOICE

  case $CHOICE in
    1)
      echo -ne "${CYAN}Enter new filename to create: ${RESET}"
      read -r NEWFILE
      if [ -e "$NEWFILE" ]; then
        echo -e "${YELLOW}File already exists.${RESET}"
      else
        touch "$NEWFILE"
        echo -e "${GREEN}File '$NEWFILE' created.${RESET}"
      fi
      ;;
    2)
      echo -ne "${CYAN}Enter filename to edit: ${RESET}"
      read -r EDITFILE
      if [ ! -e "$EDITFILE" ]; then
        echo -e "${YELLOW}File does not exist.${RESET}"
      else
        nano "$EDITFILE"
      fi
      ;;
    3)
      echo -ne "${CYAN}Enter filename to delete: ${RESET}"
      read -r DELFILE
      if [ ! -e "$DELFILE" ]; then
        echo -e "${YELLOW}File does not exist.${RESET}"
      else
        rm -i "$DELFILE"
        echo -e "${GREEN}File deleted.${RESET}"
      fi
      ;;
    4)
      git status
      echo -ne "${CYAN}Enter commit message (leave empty for default): ${RESET}"
      read -r MSG
      git add .
      if [ -z "$MSG" ]; then
        MSG="auto update from script"
      fi
      git commit -m "$MSG" && echo -e "${GREEN}Changes committed.${RESET}" || echo -e "${YELLOW}Nothing to commit.${RESET}"
      ;;
    5)
      echo -ne "${CYAN}Push changes to GitHub? (yes/no): ${RESET}"
      read -r PUSH_ANSWER
      if [ "$PUSH_ANSWER" == "yes" ]; then
        git pull --no-edit
        if git push; then
          echo -e "${GREEN}Changes pushed successfully.${RESET}"

          # ساخت فایل links.md از تغییرات آخرین commit
          echo "" > links.md
          CHANGED_FILES=$(git diff --name-only HEAD~1)

          for FILE in $CHANGED_FILES; do
            RAW_URL="https://raw.githubusercontent.com/tepo18/$REPO_NAME/main/$FILE"
            echo "$RAW_URL" >> links.md
          done

          # آخرین لینک رو تو کلیپ‌بورد ترموکس قرار میده (اگر فایل خالی نباشه)
          LAST_LINK=$(tail -n 1 links.md)
          if [ -n "$LAST_LINK" ]; then
            echo "$LAST_LINK" | termux-clipboard-set
            echo -e "${CYAN}Last Sublink copied to clipboard:${RESET} $LAST_LINK"
          fi

          echo -e "${GREEN}All sublinks saved in links.md.${RESET}"
        else
          echo -e "${YELLOW}Push failed.${RESET}"
        fi
      else
        echo "Push canceled."
      fi
      ;;
    6)
      echo -ne "${CYAN}Enter source filename inside download folder ($DOWNLOAD_DIR): ${RESET}"
      read -r SRCFILE
      SRC_PATH="$DOWNLOAD_DIR/$SRCFILE"
      if [ ! -f "$SRC_PATH" ]; then
        echo -e "${YELLOW}Source file '$SRC_PATH' not found.${RESET}"
        continue
      fi
      echo -ne "${CYAN}Enter target filename inside repo folder: ${RESET}"
      read -r TARGETFILE
      cat "$SRC_PATH" > "$TARGETFILE"
      echo -e "${GREEN}Content from '$SRC_PATH' copied to '$TARGETFILE'.${RESET}"
      ;;
    7)
      echo "Exiting."
      exit 0
      ;;
    *)
      echo -e "${YELLOW}Invalid choice!${RESET}"
      ;;
  esac
done

#!/bin/bash

CYAN="\e[36m"
GREEN="\e[32m"
YELLOW="\e[33m"
RED="\e[31m"
RESET="\e[0m"

BASE_PATH="$HOME"
DOWNLOAD_DIR="/sdcard/Download/Akbar98"

# تنظیم نام و ایمیل گیت برای کامیت‌ها (یکبار برای همیشه)
GIT_USER_NAME="Your Name"
GIT_USER_EMAIL="your_email@example.com"

# تنظیم گیت یوزر و ایمیل لوکال در ریپو
function set_git_identity() {
  git config user.name "$GIT_USER_NAME"
  git config user.email "$GIT_USER_EMAIL"
}

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
      echo -e "${RED}Failed to clone repo. Exiting.${RESET}"
      exit 1
    fi
  fi

elif [ "$CLONE_CHOICE" == "2" ] ; then
  echo -ne "${CYAN}Enter the existing cloned repo folder name: ${RESET}"
  read -r REPO_NAME
  REPO_PATH="$BASE_PATH/$REPO_NAME"
  
  if [ ! -d "$REPO_PATH" ]; then
    echo -e "${RED}Folder '$REPO_PATH' not found! Exiting.${RESET}"
    exit 1
  fi
else
  echo -e "${RED}Invalid choice. Exiting.${RESET}"
  exit 1
fi

cd "$REPO_PATH" || { echo -e "${RED}Cannot enter repo folder. Exiting.${RESET}"; exit 1; }
set_git_identity

while true; do
  echo -e "\n${CYAN}Select an option:${RESET}"
  echo "1) Create new file"
  echo "2) Edit existing file"
  echo "3) Delete a file"
  echo "4) Commit changes"
  echo "5) Push to GitHub"
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
      # Commit changes (auto if needed)
      git status --porcelain > /dev/null
      if [ $? -ne 0 ]; then
        echo -e "${RED}Git repository not initialized correctly.${RESET}"
        break
      fi

      # اگر تغییرات unstaged یا staged بود کامیت کن
      if [ -n "$(git status --porcelain)" ]; then
        echo -ne "${CYAN}Enter commit message (default: auto commit): ${RESET}"
        read -r MSG
        if [ -z "$MSG" ]; then
          MSG="auto commit from script"
        fi
        git add .
        git commit -m "$MSG"
        if [ $? -eq 0 ]; then
          echo -e "${GREEN}Changes committed.${RESET}"
        else
          echo -e "${YELLOW}Commit failed or nothing to commit.${RESET}"
        fi
      else
        echo -e "${YELLOW}No changes to commit.${RESET}"
      fi
      ;;
    5)
      echo -ne "${CYAN}Push changes to GitHub? (yes/no): ${RESET}"
      read -r PUSH_ANSWER
      if [ "$PUSH_ANSWER" == "yes" ]; then
        # ابتدا بررسی کن که آیا شاخه main یا master هست
        CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD 2>/dev/null)
        if [ -z "$CURRENT_BRANCH" ]; then
          # شاخه فعلی وجود ندارد، پس یکی بساز
          echo -e "${YELLOW}Current branch not found. Creating 'main' branch.${RESET}"
          git checkout -b main
          CURRENT_BRANCH="main"
        fi

        # سعی کن پوش کنی
        git push origin "$CURRENT_BRANCH"
        PUSH_RESULT=$?

        if [ $PUSH_RESULT -ne 0 ]; then
          echo -e "${YELLOW}Push failed. Trying to pull and merge automatically...${RESET}"

          # اگر شاخه remote موجود نبود، یکی بساز
          if ! git ls-remote --heads origin "$CURRENT_BRANCH" | grep "$CURRENT_BRANCH" > /dev/null; then
            echo -e "${YELLOW}Remote branch '$CURRENT_BRANCH' does not exist. Creating remote branch...${RESET}"
            git push -u origin "$CURRENT_BRANCH"
          else
            # اگر remote بود، pull بزن
            git pull --no-rebase origin "$CURRENT_BRANCH"
            PULL_RESULT=$?
            if [ $PULL_RESULT -eq 0 ]; then
              echo -e "${GREEN}Pulled remote changes successfully.${RESET}"
              echo -e "${CYAN}Trying to push again...${RESET}"
              git push origin "$CURRENT_BRANCH"
              SECOND_PUSH_RESULT=$?
              if [ $SECOND_PUSH_RESULT -eq 0 ]; then
                echo -e "${GREEN}Push successful after pulling.${RESET}"
              else
                echo -e "${RED}Push failed again. Please resolve conflicts manually.${RESET}"
              fi
            else
              echo -e "${RED}Pull failed. Please resolve conflicts manually.${RESET}"
            fi
          fi
        else
          echo -e "${GREEN}Changes pushed successfully.${RESET}"
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

      # Copy content exactly
      cat "$SRC_PATH" > "$TARGETFILE"
      echo -e "${GREEN}Content from '$SRC_PATH' copied to '$TARGETFILE' successfully.${RESET}"
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

---
title: "YES24 도서 크롤링 과제"
date: 2025-05-12
tags: [크롤링, R, RSelenium, yes24]
categories: [프로젝트]
---

## 📘 YES24 도서 크롤링 과제

**목표:** YES24의 에세이 카테고리에서 모든 페이지, 책 상세 정보 크롤링 하기

---

### 1. Rselenium을 이용해 모든 책을 한 번 씩 클릭하여 html로 크롤링

- ✅ 방법: 책 하나씩 클릭 후 상세 페이지 저장  
- ❌ 결과: **소요시간이 지나치게 오래 걸림**  
- 💭 생각: 이렇게 하면 며칠이 걸릴 것 같음 → **다른 방식 필요**

#### 📎 코드
<details>
<summary>코드 보기</summary>

```r
"library(RSelenium)
library(stringr)
library(readr)
library(rvest)
library(fs)

# 기본 설정
Sys.setenv(PATH = paste0(Sys.getenv("PATH"), ";C:/chromedriver/"))
port_num <- sample(4000:5000, 1)
driver <- rsDriver(browser = "chrome", port = port_num, chromever = NULL, verbose = FALSE, check = FALSE)
remDr <- driver$client

# 크롤링할 카테고리 ID 및 시작점
category_ids <- c("001001047005", "001001047007", "001001047008", "001001047009", "001001047019",
                                    "001001047003", "001001047010", "001001047011", "001001047012", "001001047013",
                                    "001001047014", "001001047018", "001001047004", "001001047015", "001001047016",
                                    "001001047017", "001001047006", "001001047002", "001001047001")

start_info <- list(
  "001001047012" = list(blocks = 2, page = 21),
  "001001047014" = list(blocks = 0, page = 7)
)

base_dir <- "C:/Users/751mb/OneDrive - 전북대학교/문서/yes24"

# 크롤링 루프 시작
for (cat_id in category_ids) {
  cat_dir <- file.path(base_dir, cat_id)
  books_dir <- file.path(cat_dir, "old_books")
  dir_create(books_dir, recurse = TRUE)
  
  block_jumps <- if (!is.null(start_info[[cat_id]])) start_info[[cat_id]]$blocks else 0
  page_num <- if (!is.null(start_info[[cat_id]])) start_info[[cat_id]]$page else 1
  
  url <- paste0("http://www.yes24.com/24/Category/Display/", cat_id)
  remDr$navigate(url)
  Sys.sleep(4)
  
  tryCatch({
    pg_size_selector <- remDr$findElement(using = "css selector", "#pg_size")
    pg_size_selector$sendKeysToElement(list("120"))
    Sys.sleep(5)
  }, error = function(e) {
    cat("⚠️ 120개 보기 실패: ", cat_id, "\n")
  })
  
  for (i in seq_len(block_jumps)) {
    next_btns <- remDr$findElements("css selector", "a.bgYUI.next")
    if (length(next_btns) > 0) {
      next_btns[[1]]$clickElement()
      Sys.sleep(3)
    }
  }
  
  if (page_num > 1) {
    page_links <- remDr$findElements("css selector", "div.yesUI_pagen a.num")
    idx <- which(sapply(page_links, function(el) el$getElementText()[[1]]) == as.character(page_num))
    if (length(idx) > 0) {
      page_links[[idx]]$clickElement()
      Sys.sleep(3)
    }
  }
  
  existing_files <- dir(books_dir, pattern = "^book_\\d+\\.html$")
  existing_uids <- na.omit(str_extract(existing_files, "\\d+"))
  
  repeat {
    page_source <- remDr$getPageSource()[[1]]
    list_file <- file.path(cat_dir, paste0("cat_", cat_id, "_list_page_", page_num, ".html"))
    write(page_source, list_file)
    cat("📄 ", list_file, " 저장됨\n")
    
    book_links <- remDr$findElements("css selector", "a.gd_name")
    for (i in seq_along(book_links)) {
      book_links <- remDr$findElements("css selector", "a.gd_name")
      if (length(book_links) < i) next
      
      tryCatch({
        book_links[[i]]$clickElement()
        Sys.sleep(3)
        
        current_url <- remDr$getCurrentUrl()[[1]]
        if (grepl("goods/(\\d+)", current_url)) {
          book_uid <- str_match(current_url, "goods/(\\d+)")[,2]
          if (book_uid %in% existing_uids) {
            cat("⏩ 중복된 책 (", book_uid, ") 건너뜀\n")
            remDr$goBack(); Sys.sleep(2)
            next
          }
          
          expand_buttons <- remDr$findElements("css selector", "a.btn_txt_more")
          for (btn in expand_buttons) {
            tryCatch({ btn$clickElement(); Sys.sleep(1) }, error = function(e) {})
          }
          
          book_file <- file.path(books_dir, paste0("book_", book_uid, ".html"))
          writeLines(remDr$getPageSource()[[1]], book_file, useBytes = TRUE)
          cat("📚 책 저장 완료 (page ", page_num, ", #", i, "): book_", book_uid, "\n", sep = "")
          
          existing_uids <- c(existing_uids, book_uid)
        }
        remDr$goBack(); Sys.sleep(2)
      }, error = function(e) {
        cat("❌ 책 저장 실패 (page ", page_num, ", 책 ", i, "): ", e$message, "\n")
        remDr$goBack(); Sys.sleep(2)
      })
    }
    
    # 페이지 이동을 title 속성 기반으로 하는 안정적인 방식으로 수정된 전체 페이지 이동 블록
    
    # 다음 페이지로 이동
    next_page <- as.character(page_num + 1)
    page_links <- remDr$findElements(using = "css selector", "div.yesUI_pagen a.num")
    link_titles <- sapply(page_links, function(el) el$getElementAttribute("title")[[1]])
    
    idx <- which(link_titles == next_page)
    
    if (length(idx) > 0) {
      tryCatch({
        page_links[[idx]]$clickElement()
        Sys.sleep(3)
        page_num <- page_num + 1
      }, error = function(e) {
        cat("페이지", next_page, "로 이동 실패. 종료\n")
        break
      })
      
    } else {
      # 다음 블록으로 넘어가기
      next_btns <- remDr$findElements(using = "css selector", "a.bgYUI.next")
      if (length(next_btns) > 0) {
        tryCatch({
          next_btns[[1]]$clickElement()
          Sys.sleep(3)
          # 다음 블록이 로드된 후, 새 페이지 번호가 있는지 다시 확인
          page_links <- remDr$findElements(using = "css selector", "div.yesUI_pagen a.num")
          link_titles <- sapply(page_links, function(el) el$getElementAttribute("title")[[1]])
          idx <- which(link_titles == next_page)
          if (length(idx) > 0) {
            page_links[[idx]]$clickElement()
            Sys.sleep(3)
            page_num <- page_num + 1
            cat("➡️ 다음 블록 + 페이지 이동 완료. 페이지:", page_num, "\n")
          } else {
            cat("🔚 다음 블록에서 페이지 못 찾음. 종료\n")
            break
          }
        }, error = function(e) {
          cat("블록 이동 실패. 종료\n")
          break
        })
      } else {
        cat("🔚 더 이상 페이지 없음. 종료\n")
        break
      }
    }
  }
}
```
</details>

### 2. Rselenium으로 여러 창 띄워서 병렬 크롤링
 
- ❌ 결과: **오류 메시지 발생**  
- 💭 생각: 병렬 작업이 더 어려울 수 있음 → 복잡도 증가


### 3. 책 페이지 HTML 저장 후 링크로 이동해 개별 HTML 저장

- ✅ 방법: 각 페이지를 저장 → 고유 ID 추출 → 링크 조립 → read_html()로 빠르게 저장 
- ❌ 결과: **가장 빠르고 안정적**  
- ⚠️ 제한: 펼쳐보기 클릭 없이 모든 정보가 나오는지 확인 필요

💡 **페이지네이션 개선**
- 기존: 10 → 11 페이지 넘어갈 때 오류 발생
- 개선: HTML 구조 분석 후 title 속성 기반으로 안정적으로 이동 가능

#### 📎 코드
<details>
<summary>코드 보기</summary>
```{r}
library(RSelenium)
library(stringr)
library(readr)
library(rvest)
library(fs)

# 기본 설정
Sys.setenv(PATH = paste0(Sys.getenv("PATH"), ";C:/chromedriver/"))
port_num <- sample(4000:5000, 1)
driver <- rsDriver(browser = "chrome", port = port_num, chromever = NULL, verbose = FALSE, check = FALSE)
remDr <- driver$client

# 카테고리 및 시작점
category_ids <- c("001001047005", "001001047007", "001001047008", "001001047009", "001001047019",
                  "001001047003", "001001047010", "001001047011", "001001047012", "001001047013",
                  "001001047014", "001001047018", "001001047004", "001001047015", "001001047016",
                  "001001047017", "001001047006", "001001047002", "001001047001")

start_info <- list(
  "001001047002" = list(blocks = 3, page = 31)
)


for (cat_id in category_ids) {
  block_jumps <- if (!is.null(start_info[[cat_id]])) start_info[[cat_id]]$blocks else 0
  page_num <- if (!is.null(start_info[[cat_id]])) start_info[[cat_id]]$page else 1
  
  list_dir <- file.path("C:/Users/751mb/OneDrive - 전북대학교/문서/yes24", cat_id, "list")
  dir_create(list_dir, recurse = TRUE)
  
  url <- paste0("http://www.yes24.com/24/Category/Display/", cat_id)
  remDr$navigate(url)
  Sys.sleep(4)
  
  tryCatch({
    pg_size_selector <- remDr$findElement(using = "css selector", "#pg_size")
    pg_size_selector$sendKeysToElement(list("120"))
    Sys.sleep(5)
  }, error = function(e) {
    cat("⚠️ 120개 보기 실패 (", cat_id, "): ", e$message, "\n")
  })
  
  for (i in seq_len(block_jumps)) {
    next_btns <- remDr$findElements(using = "css selector", "a.bgYUI.next")
    if (length(next_btns) > 0) {
      next_btns[[1]]$clickElement()
      Sys.sleep(3)
    }
  }
  
  if (page_num > 1) {
    page_links <- remDr$findElements("css selector", "div.yesUI_pagen a.num")
    idx <- which(sapply(page_links, function(el) el$getElementText()[[1]]) == as.character(page_num))
    if (length(idx) > 0) {
      page_links[[idx]]$clickElement()
      Sys.sleep(4)
    }
  }
  
  repeat {
    # ✅ 현재 페이지 HTML 저장
    page_source <- remDr$getPageSource()[[1]]
    list_file <- file.path(list_dir, paste0("cat_", cat_id, "_list_page_", page_num, ".html"))
    write(page_source, file = list_file)
    cat("✅ 카테고리", cat_id, "페이지", page_num, "저장 완료 →", list_file, "\n")
    
    next_page <- as.character(page_num + 1)
    
    # 1. 일반 번호 페이지 링크 먼저 탐색
    page_links <- remDr$findElements(using = "css selector", "div.yesUI_pagen a.num")
    link_titles <- sapply(page_links, function(el) el$getElementAttribute("title")[[1]])
    idx <- which(link_titles == next_page)
    
    if (length(idx) > 0) {
      tryCatch({
        page_links[[idx]]$clickElement()
        Sys.sleep(3)
        page_num <- page_num + 1
      }, error = function(e) {
        cat("페이지", next_page, "로 이동 실패. 종료\n")
        break
      })
    } else {
      # 2. 다음 블록 버튼의 title이 다음 페이지 번호인지 확인
      next_btns <- remDr$findElements(using = "css selector", "a.bgYUI.next")
      if (length(next_btns) > 0) {
        next_titles <- sapply(next_btns, function(el) el$getElementAttribute("title")[[1]])
        idx_next <- which(next_titles == next_page)
        
        if (length(idx_next) > 0) {
          tryCatch({
            next_btns[[idx_next]]$clickElement()
            Sys.sleep(4)
            page_num <- page_num + 1
            cat("➡️ 다음 블록으로 이동: 페이지", page_num, "\n")
          }, error = function(e) {
            cat("❌ 다음 블록 클릭 실패. 종료\n")
            break
          })
        } else {
          cat("🔚 다음 블록 없음. 종료\n")
          break
        }
      } else {
        cat("🔚 더 이상 페이지 없음. 종료\n")
        break
      }
    }
  }
  
} # for 끝
```  
</details>
📎 코드 - 링크 통해 책 HTML 저장
<details> <summary>코드 보기</summary>
```{r}
library(rvest)
library(stringr)
library(fs)

# 1. 전체 카테고리 경로 설정
base_dir <- "C:/Users/751mb/OneDrive - 전북대학교/문서/yes24"
category_ids <- c("001001047005", "001001047007", "001001047008", "001001047009", "001001047019",
                  "001001047003", "001001047010", "001001047011", "001001047012", "001001047013",
                  "001001047014", "001001047018", "001001047004", "001001047015", "001001047016",
                  "001001047017", "001001047006", "001001047002", "001001047001")
)

library(rvest)
library(stringr)
library(fs)

# 1. 전체 카테고리 경로 설정
base_dir <- "C:/Users/751mb/OneDrive - 전북대학교/문서/yes24"
category_ids <- c("001001047001")  # 두 카테고리

# 2. 각 카테고리 순회
for (cat_id in category_ids) {
  list_dir <- file.path(base_dir, cat_id, "list")
  save_dir <- file.path(base_dir, cat_id, "new_books")
  dir_create(save_dir, recurse = TRUE)
  
  # 예외 페이지 목록 (비워둠)
  excluded_pages <- c()
  excluded_pattern <- paste0("_page_(", paste(excluded_pages, collapse = "|"), ")\\.html$")
  
  # list 파일 필터링
  list_files <- dir_ls(list_dir, glob = "*.html")
  if (length(excluded_pages) > 0) {
    list_files <- list_files[!str_detect(list_files, excluded_pattern)]
  }
  
  saved_pages <- c()  # 페이지 누적 추적용
  
  for (list_path in list_files) {
    page_num <- str_match(basename(list_path), "_page_(\\d+)\\.html")[,2]
    
    # ✅ 새 페이지 감지 시 출력
    if (!(page_num %in% saved_pages)) {
      saved_pages <- c(saved_pages, page_num)
      saved_pages <- sort(as.numeric(saved_pages))
      cat(sprintf("📄 저장된 페이지: (%s)\n", paste(saved_pages, collapse = ", ")))
    }
    
    list_html <- tryCatch(read_html(list_path), error = function(e) NULL)
    if (is.null(list_html)) {
      cat("⚠️ HTML 로드 실패:", list_path, "\n")
      next
    }
    
    book_links <- list_html %>%
      html_elements("a.gd_name") %>%
      html_attr("href")
    
    book_uids <- str_match(book_links, "/product/goods/(\\d+)")[,2]
    book_uids <- na.omit(book_uids)
    
    for (i in seq_along(book_uids)) {
      book_uid <- book_uids[i]
      save_path <- file.path(save_dir, paste0("book_", book_uid, ".html"))
      
      if (file_exists(save_path)) {
        cat("⏩ 이미 저장됨:", book_uid, "\n")
        next
      }
      
      book_url <- paste0("https://www.yes24.com/Product/Goods/", book_uid)
      
      tryCatch({
        Sys.sleep(1)
        html <- read_html(book_url)
        writeLines(as.character(html), save_path, useBytes = TRUE)
        cat(sprintf("[📂%s] 📄 %spg | 📘 %02d번째 책 | UID: %s\n",
                    cat_id, page_num, i, book_uid))
      }, error = function(e) {
        cat(sprintf("❌ 저장 실패: [카테고리 %s] %s페이지의 %d번째 책 (book_uid: %s) - %s\n",
                    cat_id, page_num, i, book_uid, e$message))
      })
    }
  }
}
```
</details>

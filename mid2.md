2번 문제
```
#include <stdio.h>
#include <stdlib.h>
#include <sqlite3.h>

static int print_row(void *u, int n, char **v, char **c)
{
    (void)u; (void)c;
    for (int i = 0; i < n; i++)
        printf("%s%s", v[i] ? v[i] : "NULL", i < n - 1 ? " | " : "\n");
    return 0;
}

static void run(sqlite3 *db, const char *sql)
{
    char *err = NULL;
    if (sqlite3_exec(db, sql, print_row, NULL, &err) != SQLITE_OK) {
        fprintf(stderr, "SQL 오류: %s\n", err);
        sqlite3_free(err);
        exit(1);
    }
}

static void search_books(sqlite3 *db, const char *kw)
{
    const char *sql =
        "SELECT b.id, b.title, b.author, b.year, "
        "CASE WHEN l.id IS NULL THEN '대출가능' ELSE '대출중(' || m.name || ')' END "
        "FROM books b "
        "LEFT JOIN loans l ON l.book_id = b.id AND l.returned = 0 "
        "LEFT JOIN members m ON m.id = l.member_id "
        "WHERE b.title LIKE ?1 OR b.author LIKE ?1 "
        "ORDER BY b.id;";
    sqlite3_stmt *st;
    char pat[256];
    snprintf(pat, sizeof(pat), "%%%s%%", kw);

    if (sqlite3_prepare_v2(db, sql, -1, &st, NULL) != SQLITE_OK) {
        fprintf(stderr, "%s\n", sqlite3_errmsg(db));
        exit(1);
    }
    sqlite3_bind_text(st, 1, pat, -1, SQLITE_TRANSIENT);

    int found = 0;
    while (sqlite3_step(st) == SQLITE_ROW) {
        printf("%d | %s | %s | %d | %s\n",
               sqlite3_column_int(st, 0),
               sqlite3_column_text(st, 1),
               sqlite3_column_text(st, 2),
               sqlite3_column_int(st, 3),
               sqlite3_column_text(st, 4));
        found++;
    }
    if (!found) printf("검색 결과 없음\n");
    sqlite3_finalize(st);
}

int main(int argc, char *argv[])
{
    const char *kw = argc > 1 ? argv[1] : "C";
    sqlite3 *db;

    remove("library.db");
    if (sqlite3_open("library.db", &db) != SQLITE_OK) return 1;

    run(db,
        "PRAGMA foreign_keys = ON;"
        "CREATE TABLE members("
        "  id INTEGER PRIMARY KEY AUTOINCREMENT,"
        "  name TEXT NOT NULL,"
        "  dept TEXT"
        ");"
        "CREATE TABLE books("
        "  id INTEGER PRIMARY KEY AUTOINCREMENT,"
        "  title TEXT NOT NULL,"
        "  author TEXT,"
        "  year INTEGER"
        ");"
        "CREATE TABLE loans("
        "  id INTEGER PRIMARY KEY AUTOINCREMENT,"
        "  member_id INTEGER NOT NULL REFERENCES members(id),"
        "  book_id INTEGER NOT NULL REFERENCES books(id),"
        "  loan_date TEXT NOT NULL,"
        "  returned INTEGER NOT NULL DEFAULT 0"
        ");");

    run(db,
        "INSERT INTO members(name, dept) VALUES"
        "  ('김민수', '컴퓨터공학'),"
        "  ('이서연', '수학'),"
        "  ('박지훈', '전자공학');"
        "INSERT INTO books(title, author, year) VALUES"
        "  ('C 프로그래밍 언어', 'Kernighan', 1988),"
        "  ('이산수학', 'Rosen', 2019),"
        "  ('파이썬 입문', '김영수', 2021),"
        "  ('알고리즘 개론', 'Cormen', 2022);"
        "INSERT INTO loans(member_id, book_id, loan_date, returned) VALUES"
        "  (1, 1, '2026-10-01', 0),"
        "  (2, 2, '2026-10-05', 0),"
        "  (3, 1, '2026-09-10', 1);");

    printf("[회원 목록]\n");
    run(db, "SELECT * FROM members;");

    printf("\n[도서 목록]\n");
    run(db, "SELECT * FROM books;");

    printf("\n[대출 현황 (JOIN)]\n");
    run(db,
        "SELECT m.name, b.title, l.loan_date, "
        "CASE l.returned WHEN 1 THEN '반납' ELSE '대출중' END "
        "FROM loans l JOIN members m ON m.id = l.member_id "
        "JOIN books b ON b.id = l.book_id ORDER BY l.loan_date;");

    printf("\n[검색: \"%s\"]\n", kw);
    search_books(db, kw);

    sqlite3_close(db);
    return 0;
}
```

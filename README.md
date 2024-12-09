#include "crow_all.h"
#include <sqlite3.h>
#include <string>
#include <iostream>

// Инициализация базы данных
void initializeDatabase(sqlite3* &db) {
    const char* createTableQuery =
        "CREATE TABLE IF NOT EXISTS attendance ("
        "id INTEGER PRIMARY KEY AUTOINCREMENT,"
        "student_name TEXT NOT NULL,"
        "present INTEGER NOT NULL);"
        "CREATE TABLE IF NOT EXISTS grades ("
        "id INTEGER PRIMARY KEY AUTOINCREMENT,"
        "student_name TEXT NOT NULL,"
        "subject TEXT NOT NULL,"
        "grade INTEGER NOT NULL);";

    char* errorMessage;
    if (sqlite3_exec(db, createTableQuery, 0, 0, &errorMessage) != SQLITE_OK) {
        std::cerr << "Failed to initialize database: " << errorMessage << std::endl;
        sqlite3_free(errorMessage);
    }
}

int main() {
    // Подключаемся к базе данных
    sqlite3* db;
    if (sqlite3_open("school.db", &db)) {
        std::cerr << "Can't open database: " << sqlite3_errmsg(db) << std::endl;
        return 1;
    }
    initializeDatabase(db);

    crow::SimpleApp app;

    // Маршрут для добавления посещаемости
    CROW_ROUTE(app, "/attendance").methods("POST"_method)([&](const crow::request& req) {
        auto body = crow::json::load(req.body);
        if (!body || !body.has("student_name") || !body.has("present")) {
            return crow::response(400, "Invalid input");
        }

        std::string studentName = body["student_name"].s();
        int present = body["present"].i();

        std::string sql = "INSERT INTO attendance (student_name, present) VALUES ('" +
                          studentName + "', " + std::to_string(present) + ");";
        char* errorMessage;
        if (sqlite3_exec(db, sql.c_str(), 0, 0, &errorMessage) != SQLITE_OK) {
            std::cerr << "Database error: " << errorMessage << std::endl;
            sqlite3_free(errorMessage);
            return crow::response(500, "Database error");
        }

        return crow::response(200, "Attendance added");
    });

    // Маршрут для добавления оценки
    CROW_ROUTE(app, "/grades").methods("POST"_method)([&](const crow::request& req) {
        auto body = crow::json::load(req.body);
        if (!body || !body.has("student_name") || !body.has("subject") || !body.has("grade")) {
            return crow::response(400, "Invalid input");
        }

        std::string studentName = body["student_name"].s();
        std::string subject = body["subject"].s();
        int grade = body["grade"].i();

        std::string sql = "INSERT INTO grades (student_name, subject, grade) VALUES ('" +
                          studentName + "', '" + subject + "', " + std::to_string(grade) + ");";
        char* errorMessage;
        if (sqlite3_exec(db, sql.c_str(), 0, 0, &errorMessage) != SQLITE_OK) {
            std::cerr << "Database error: " << errorMessage << std::endl;
            sqlite3_free(errorMessage);
            return crow::response(500, "Database error");
        }

        return crow::response(200, "Grade added");
    });

    // Маршрут для просмотра посещаемости
    CROW_ROUTE(app, "/attendance/<string>").methods("GET"_method)([&](const crow::request&, std::string studentName) {
        std::string sql = "SELECT present FROM attendance WHERE student_name = '" + studentName + "';";
        sqlite3_stmt* stmt;
        if (sqlite3_prepare_v2(db, sql.c_str(), -1, &stmt, 0) != SQLITE_OK) {
            return crow::response(500, "Database error");
        }

        crow::json::wvalue result;
        int presentSum = 0, totalRecords = 0;

        while (sqlite3_step(stmt) == SQLITE_ROW) {
            presentSum += sqlite3_column_int(stmt, 0);
            ++totalRecords;
        }
        sqlite3_finalize(stmt);

        result["student_name"] = studentName;
        result["attendance"] = totalRecords > 0 ? (presentSum * 100 / totalRecords) : 0;

        return crow::response(result);
    });

    // Маршрут для просмотра оценок
    CROW_ROUTE(app, "/grades/<string>").methods("GET"_method)([&](const crow::request&, std::string studentName) {
        std::string sql = "SELECT subject, grade FROM grades WHERE student_name = '" + studentName + "';";
        sqlite3_stmt* stmt;
        if (sqlite3_prepare_v2(db, sql.c_str(), -1, &stmt, 0) != SQLITE_OK) {
            return crow::response(500, "Database error");
        }

        crow::json::wvalue result;
        crow::json::wvalue grades = crow::json::wvalue::list();

        while (sqlite3_step(stmt) == SQLITE_ROW) {
            crow::json::wvalue gradeRecord;
            gradeRecord["subject"] = reinterpret_cast<const char*>(sqlite3_column_text(stmt, 0));
            gradeRecord["grade"] = sqlite3_column_int(stmt, 1);
            grades.push_back(gradeRecord);
        }
        sqlite3_finalize(stmt);

        result["student_name"] = studentName;
        result["grades"] = grades;

        return crow::response(result);
    });

    // Запуск сервера
    app.port(8080).multithreaded().run();

    // Закрываем базу данных при завершении работы
    sqlite3_close(db);

    return 0;
}

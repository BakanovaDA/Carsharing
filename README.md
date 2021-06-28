// Предметная область: Обслуживание клиентов в бюро проката автомобилей
// Метод хеширования: Закрытое хеширование с линейным опробованием
// Метод сортировки: Слиянием
// Вид списка: Линейный однонаправленный
// Метод обхода дерева: Обратный
// Алгоритм поиска слова в тексте: Прямой

// Для обнаружения утечек памяти
#define _CRTDBG_MAP_ALLOC
#include <stdlib.h>
#include <crtdbg.h>
#ifdef _DEBUG
#ifndef DBG_NEW
#define DBG_NEW new ( _NORMAL_BLOCK , __FILE__ , __LINE__ )
#define newDBG_NEW
#endif
#endif

#include <iostream>
#include <Windows.h>
#include <fstream>
#include <sstream>
#include <iomanip> // для функции setw(n)
#define HASH_TABLE_SIZE 100
using namespace std;

/*************************************************************************************************/
/***************************** данные о КЛИЕНТАХ (АВЛ-дерево поиска) *****************************/
/*************************************************************************************************/

#pragma region
struct Client // данные о клиенте
{
    string drivers_license; // водительское удостоверение (номер)
    string FIO;
    string passport_details; // паспортные данные
    string address;
};

struct Tree_Node
{
    Client data; // данные о клиенте
    int height; // высота дерева
    // указатели на левого и правого потомка
    Tree_Node* left;
    Tree_Node* right;
    // конструктор для создания нового узла дерева
    Tree_Node(Client new_client) { data = new_client; left = right = 0; height = 1; }
};

// обертка для height
// получаем высоту вершины
int height(Tree_Node* p)
{
    if (p)
        return p->height;
    else
        return 0;
}

// вычисляем balance factor узла
// работает только с ненулевыми указателями
int bfactor(Tree_Node* p)
{
    return height(p->right) - height(p->left);
}

// восстанавливает корректное значение поля height заданного узла,
// если была нарушена сбалансированность
void fix_height(Tree_Node* p)
{
    int hr = height(p->right);
    int hl = height(p->left);
    if (hl > hr)
        p->height = hl + 1;
    else
        p->height = hr + 1;
}

// правый поворот вокруг p
Tree_Node* rotate_right(Tree_Node* p)
{
    Tree_Node* q = p->left;
    p->left = q->right;
    q->right = p;
    fix_height(p);
    fix_height(q);
    return q;
}

// левый поворот вокруг q
Tree_Node* rotate_left(Tree_Node* q)
{
    Tree_Node* p = q->right;
    q->right = p->left;
    p->left = q;
    fix_height(q);
    fix_height(p);
    return p;
}

// балансировка
Tree_Node* balance(Tree_Node* p)
{
    fix_height(p);
    if (bfactor(p) == 2)
    {
        if (bfactor(p->right) < 0)
            p->right = rotate_right(p->right);
        return rotate_left(p);
    }
    if (bfactor(p) == -2)
    {
        if (bfactor(p->left) > 0)
            p->left = rotate_left(p->left);
        return rotate_right(p);
    }
    //балансировка не нужна
    return p;
}

// ф-ция проверки на наличие клиента в БД
void find_client(Tree_Node* p, string drivers_license, bool& found)
{
    if (!p)
        return;
    if (p->data.drivers_license == drivers_license)
    {
        found = true;
        return;
    }
    else
        for (int i = 0; i < drivers_license[i]; i++)
        {
            if (drivers_license[i] > p->data.drivers_license[i])
                find_client(p->right, drivers_license, found);
            else if (drivers_license[i] < p->data.drivers_license[i])
                find_client(p->left, drivers_license, found);
            else
                continue;
        }
}

// ф-ция добавления нового клиента
Tree_Node* insert_client(Tree_Node* p, Client new_client)
{
    if (!p)
        return new Tree_Node(new_client);
    for (int i = 0; ; i++)
    {
        if (new_client.drivers_license[i] > p->data.drivers_license[i])
            p->right = insert_client(p->right, new_client);
        else if (new_client.drivers_license[i] < p->data.drivers_license[i])
            p->left = insert_client(p->left, new_client);
        else
            continue;
        return balance(p);
    }
}

// ф-ция поиска узла с min значением в дереве
Tree_Node* find_min_knot(Tree_Node* p)
{
    if (p->left)
        return find_min_knot(p->left);
    else
        return p;
}

// ф-ция удаления узла с min значением
Tree_Node* delete_min_knot(Tree_Node* p)
{
    if (p->left == 0)
        return p->right;
    p->left = delete_min_knot(p->left);
    return balance(p);
}

// ф-ция удаления значения, введенного с клавиатуры
Tree_Node* delete_client(Tree_Node* p, string data_to_delete)
{
    if (!p)
        return 0;
    if (p->data.drivers_license == data_to_delete)
    {
        Tree_Node* q = p->left;
        Tree_Node* r = p->right;
        delete p;
        if (!r)
            return q;
        Tree_Node* min = find_min_knot(r);
        min->right = delete_min_knot(r);
        min->left = q;
        return balance(min);
    }
    else
        for (int i = 0; i < data_to_delete.length(); i++)
        {
            if (data_to_delete[i] > p->data.drivers_license[i])
                p->right = delete_client(p->right, data_to_delete);
            else if (data_to_delete[i] < p->data.drivers_license[i])
                p->left = delete_client(p->left, data_to_delete);
            else
                continue;
        }
    return balance(p);
}

// вывод одного клиента
void print_client(Tree_Node* p)
{
    if (p)
    {
        cout << "| " << p->data.FIO << setw(35 - p->data.FIO.length()) << " | ";
        cout << setw(12) << p->data.passport_details << " | ";
        cout << setw(12) << p->data.drivers_license << "  | ";
        cout << p->data.address << setw(40 - p->data.address.length()) << " |\n";
    }
}

// вывод всех клиентов в обратном порядке обхода
void reverse_crawl(Tree_Node* p)
{
    if (p)
    {
        reverse_crawl(p->left);
        reverse_crawl(p->right);
        print_client(p);
    }
}

// поиск клиента по № вод удостоверения
void drivers_license_search(Tree_Node* p, string drivers_license, string number, int& count_print, Client& client)
{ // count_print - признак напечатанности клиента(передаем = 1 - отсутствует, = 2 - если успешный поиск), чтобы не печаталось много раз за счет рекурсии
    if (!p)
        return;
    if (p->data.drivers_license == drivers_license && count_print == 1)
    {
        cout << "| " << p->data.FIO << setw(35 - p->data.FIO.length()) << " | ";
        cout << setw(12) << p->data.passport_details << " | ";
        cout << setw(12) << p->data.drivers_license << "  | ";
        cout << p->data.address << setw(40 - p->data.address.length()) << " | ";
        cout << number << "  |\n";
        client = p->data;
        count_print++;
        return;
    }
    else
        for (int i = 0; i < drivers_license[i]; i++)
        {
            if (drivers_license[i] > p->data.drivers_license[i])
                drivers_license_search(p->right, drivers_license, number, count_print, client);
            else if (drivers_license[i] < p->data.drivers_license[i])
                drivers_license_search(p->left, drivers_license, number, count_print, client);
            else
                continue;
        }
}

// поиск клиента по фрагментам ФИО и адреса
bool direct_search_algorithm(string base, string fragment)
{
    int s1 = base.length();
    int s2 = fragment.length();
    for (int i = 0; i < s1 - s2 + 1; i++)
        for (int j = 0; j < s2; j++)
        {
            if (fragment[j] != base[i + j])
                break;
            else if (j == s2 - 1)
                return true;
        }
    return false;
}

// ф-ция перебирает всех клиентов БД
void bypassing_clients(Tree_Node* p, string fragment, bool& found)
{
    if (!p)
        return;
    if (direct_search_algorithm(p->data.FIO, fragment) || direct_search_algorithm(p->data.address, fragment))
    {
        cout << "| " << p->data.FIO << setw(41 - p->data.FIO.length()) << " | ";
        cout << setw(12) << p->data.drivers_license << "  | ";
        cout << p->data.address << setw(40 - p->data.address.length()) << " |\n";
        found = true;
    }
    bypassing_clients(p->left, fragment, found);
    bypassing_clients(p->right, fragment, found);
}
#pragma endregion

/*************************************************************************************************/
/****************************** данные об АВТОМОБИЛЯХ (хеш-таблица) ******************************/
/*************************************************************************************************/

#pragma region
struct Car // данные об автомобилях
{
    string number; // гос регистрационный номер
    string brand; // марка
    string color; // цвет
    int year_of_manufacture; // год выпуска
    bool indication_of_availability; // признак наличия
};

// хеш-функция
int hash_position(string key)
{
    int hash = 0;
    for (unsigned int i = 0; i < key.length(); i++)
        hash += int(key[i]) * int(key[i]);
    return hash % HASH_TABLE_SIZE;
}

// вывод одного автомобиля
void print_car(Car car)
{
    if (car.number != "NaN")
    {
        cout << "| " << car.number << " | " << car.brand << setw(10 - car.brand.length()) << " | " << car.color << setw(10 - car.color.length()) << " | " << car.year_of_manufacture << " | ";
        if (car.indication_of_availability)
            cout << "есть в наличии |\n";
        else
            cout << "нет в наличии  |\n";
    }
}

// вывод всех автомобилей
void print_table(Car* hash_table)
{
    for (int i = 0; i < HASH_TABLE_SIZE; i++)
        if (hash_table[i].number != "NaN")
            print_car(hash_table[i]);
}

// вывод свободных автомобилей
void print_free_car(Car* hash_table)
{
    cout << "\nСписок свободных автомобилей:\n\n";
    cout << "| " << setw(7) << "Номер" << setw(32) << " |  Марка  |  Цвет   | Год  | " << setw(11) << "Наличие" << setw(6) << " |\n";
    cout << "|-----------|---------|---------|------|----------------|\n";
    for (int i = 0; i < HASH_TABLE_SIZE; i++)
        if (hash_table[i].number != "NaN" && hash_table[i].indication_of_availability == true)
            print_car(hash_table[i]);
}

// ф-ция добавления нового автомобиля
void push_car(Car* hash_table, Car new_car)
{
    int key = hash_position(new_car.number);
    if (hash_table[key].number == "NaN")
    {
        hash_table[key] = new_car;
        return;
    }
    else
    {
        int i = 1;
        while (hash_table[key].number != "NaN")
            key = (key + 2 * i) % HASH_TABLE_SIZE;
    }
    hash_table[key] = new_car;
}

// ф-ция поиска автомобиля по номеру
Car find_car(Car* hash_table, string number, bool indication_of_availability = 0)
{
    int pos = hash_position(number);
    if (hash_table[pos].number != "NaN" && hash_table[pos].number == number)
    {
        hash_table[pos].indication_of_availability = true;
        if (indication_of_availability)
            hash_table[pos].indication_of_availability = false;
        return hash_table[pos];
    }
    else
    {
        int i = 1;
        while (i != HASH_TABLE_SIZE)
        {
            pos = (pos + 2 * i) % HASH_TABLE_SIZE;
            if (hash_table[pos].number == number)
            {
                if (indication_of_availability)
                    hash_table[pos].indication_of_availability = false;
                return hash_table[pos];
            }
            i++;
        }
    }
    Car car = { "NaN", "NaN", "NaN", 0, false };
    return car;
}

// ф-ция удаления автомобиля
void delete_car(Car* hash_table, string number)
{
    bool result = true;
    int pos = hash_position(number);
    if (hash_table[pos].number != "NaN" && hash_table[pos].number == number)
        hash_table[pos] = { "NaN", "NaN", "NaN", 0, false };
    else
    {
        int i = 1;
        while (i != HASH_TABLE_SIZE)
        {
            pos = (pos + 2 * i) % HASH_TABLE_SIZE;
            if (hash_table[pos].number == number)
            {
                hash_table[pos] = { "NaN", "NaN", "NaN", 0, false };
                break;
            }
            i++;
        }
    }
    if (!result)
        cout << "Данной машины нет в БД." << endl;
}

// поиск автомобиля по марке
void search_brand(Car* hash_table, string brand)
{
    for (int i = 0; i < HASH_TABLE_SIZE; i++)
    {
        if (brand == hash_table[i].brand)
            print_car(hash_table[i]);
    }
}
#pragma endregion

/*************************************************************************************************/
/***************************** данные об АРЕНДЕ АВТОМОБИЛЕЙ (список) *****************************/
/*************************************************************************************************/

#pragma region
struct Rent_a_Car // данные о выдаче на прокат и возврате автомобилей
{
    string drivers_license; // водительское удостоверение (номер)
    string state_registration_number; // гос регистрационный номер
    string data_of_renting; // дата выдачи
    string data_of_returning; // дата возврата
};

struct List_Node
{
    Rent_a_Car rental_inform;
    List_Node* next;
};

// регистрация выдачи клиенту автомобиля на прокат
List_Node* insert_rental_inform(List_Node*& root, Rent_a_Car rental_inform)
{
    List_Node* node = new List_Node;
    node->next = root;
    root = node; // добавленный узел стал корнем списка
    root->rental_inform = rental_inform;
    return(root);
}

string check_series_and_number(); // определяю функцию для кода ниже (457)
// регистрация возврата автомобиля от клиентов
void delete_rental_inform(Tree_Node* p, Car* hash_table, List_Node*& root)
{
    cout << "\nВведите № водительского удостоверения клиента, возвращающего автомобиль >> ";
    string drivers_license = check_series_and_number();
    // поиск записи об аренде в списке
    List_Node* ptr = root;
    while (ptr)
    {
        if (ptr->rental_inform.drivers_license == drivers_license)
        {
            // удаляем клиента из БД
            delete_client(p, drivers_license);
            // помечаем свободость автомобиля
            find_car(hash_table, ptr->rental_inform.state_registration_number);
            // удаляем запись об аренде в списке
            if (ptr == root)
            {
                root = root->next;
                delete(ptr);
                return;
            }
            else
            {
                List_Node* temp = root;
                while (temp->next != ptr) // пока не найден ключ, предшествующий 1му
                    temp = temp->next;
                temp->next = ptr->next; // переставляем указатель
                delete(ptr);
                return;
            }
        }
        ptr = ptr->next;
    }
    cout << "Водитель не найден.\n";
}

// вывод списка арендованных автобомилей и данных об аренде
void print_rental_inform(List_Node* root)
{
    List_Node* ptr = root;
    while (ptr)
    {
        cout << "| " << setw(12) << ptr->rental_inform.drivers_license << "  | ";
        cout << ptr->rental_inform.state_registration_number << " | ";
        cout << ptr->rental_inform.data_of_renting << " | ";
        cout << ptr->rental_inform.data_of_returning << " |\n";
        ptr = ptr->next;
    }
}

// слияние списков
List_Node* merge(List_Node* l, List_Node* r)
{
    List_Node* head = NULL;
    if (!l)
        return r;
    if (!r)
        return l;
    while (l && r)
        if (l->rental_inform.drivers_license < r->rental_inform.drivers_license)
        {
            head = l;
            head->next = merge(l->next, r);
            break;
        }
        else
        {
            head = r;
            head->next = merge(l, r->next);
            break;
        }
    return head;
}

// cортировка слиянием 
List_Node* merge_sort(List_Node* root)
{
    if (!root || !root->next) // список пуст, или состоит из одного элемента
        return root;
    List_Node* l = root, * r = root->next;
    while (r && r->next)
    {
        root = root->next;
        r = r->next->next;
    }
    r = root->next;
    root->next = NULL;
    return merge(merge_sort(l), merge_sort(r));
}

// поиск записи по № вод удостоверения
List_Node* search_rental_inform_drivers_license(List_Node* root, string drivers_license)
{
    List_Node* temp = root;
    while (temp)
    {
        if (temp->rental_inform.drivers_license == drivers_license)
            return temp;
        temp = temp->next;
    }
}

// поиск записи по гос номеру автомобиля
List_Node* search_rental_inform_number(List_Node* root, string number)
{
    List_Node* temp = root;
    while (temp)
    {
        if (temp->rental_inform.state_registration_number == number)
            return temp;
        temp = temp->next;
    }
}
#pragma endregion

/*************************************** функции проверки ****************************************/
#pragma region
// ф-ция обнаружения утечек памяти
void check_leak()
{
    _CrtSetReportMode(_CRT_WARN, _CRTDBG_MODE_FILE);
    _CrtSetReportFile(_CRT_WARN, _CRTDBG_FILE_STDOUT);
    _CrtSetReportMode(_CRT_ERROR, _CRTDBG_MODE_FILE);
    _CrtSetReportFile(_CRT_ERROR, _CRTDBG_FILE_STDOUT);
    _CrtSetReportMode(_CRT_ASSERT, _CRTDBG_MODE_FILE);
    _CrtSetReportFile(_CRT_ASSERT, _CRTDBG_FILE_STDOUT);
    _CrtDumpMemoryLeaks();
}

// ф-ция обнаружения ошибок при открытии файла
void check_file(fstream& file)
{
    if (!file.is_open())
    {
        cout << "Ошибка при открытии файла.";
        exit(0);
    }
}

// проверка ввода пункта меню
int check_menu()
{
    int menu;
    while (1)
    {
        cin >> menu;
        if (cin.fail()) // проверка на ошибку при вводе
        {
            cin.clear(); // возвращаем cin в "обычный" режим работы
            cin.ignore((numeric_limits<streamsize>::max)(), '\n'); // удаление значения предыдущего ввода из входного буфера
            cout << "Некорректный тип данных. Повторите ввод >> ";
        }
        else if (cin.get() != '\n') // проверка на наличие буквы или символа
        {
            cin.ignore((numeric_limits<streamsize>::max)(), '\n'); // удаление значения предыдущего ввода из входного буфера
            cout << "Некорректный тип данных. Повторите ввод >> ";
        }
        else return menu;
    }
}

// проверка ввода FIO
string check_FIO()
{
    string temp;
    while (1)
    {
        cin >> temp;
        bool flag = true;
        if (temp[0] >= 'А' && temp[0] <= 'Я')
        { // проверка на заглавие 1ой буквы
            for (int i = 1; i < temp.length(); i++)
            { // допустимые символы: только рус алфавит
                if (temp[i] > 'я' || temp[i] < 'а')
                {
                    cout << "Ошибка. Введите заново >> ";
                    flag = false;
                    break;
                }
            }
        }
        else
        {
            cout << "ФИО должны начинаться с большой буквы. Введите заново >> ";
            flag = false;
        }
        if (flag) return temp;
    }
}

// проверка ввода серии и номера
string check_series_and_number()
{
    string temp;
    cin.ignore(cin.rdbuf()->in_avail());
    while (1)
    {
        getline(cin, temp);
        bool flag = true;
        if (temp.length() == 12)
        { // проверка на длину вводимых данных
            for (int i = 0; i < temp.length(); i++)
            { // допустимые символы: только цифры
                if (i == 2 || i == 5)
                {
                    if (temp[i] != 32)
                    {
                        cout << "Серия и номер введены некорректно. Введите заново >> ";
                        flag = false;
                        break;
                    }
                }
                else
                    if (temp[i] > 57 || temp[i] < 48)
                    {
                        cout << "Серия и номер введены некорректно. Введите заново >> ";
                        flag = false;
                        break;
                    }
            }
        }
        else
        {
            cout << "Серия и номер введены некорректно. Введите заново >> ";
            flag = false;
        }
        if (flag) return temp;
    }
}

// проверка ввода адреса
string check_address()
{
    string address;
    cin.ignore(cin.rdbuf()->in_avail());
    while (1)
    {
        getline(cin, address);
        bool flag = true;
        for (int i = 0; i < address.length(); i++)
        { // допустимые символы: рус алфавит, цифры, '/', '-', ' '
            if ((address[i] >= 'А' && address[i] <= 'я') || (address[i] >= '0' && address[i] <= '9') || \
                address[i] == '-' || address[i] == '/' || address[i] == ' ')
            {
                continue;
            }
            else
            {
                cout << "Ошибка. Введите заново >> ";
                flag = false;
                break;
            }
        }
        if (flag) return address;
    }
}

// проверка ввода данных о клиенте
Client check_client(string drivers_license = "0")
{
    Client client;
    string FIO, F, I, O;
    cout << "\nВведите фамилию клиента >> "; F = check_FIO();
    cout << "\nВведите имя клиента >> "; I = check_FIO();
    cout << "\nВведите отчество клиента >> "; O = check_FIO();
    FIO = F + " " + I + " " + O;
    cout << "\nВведите паспортные данные >> ";
    string passport_details = check_series_and_number();
    if (drivers_license == "0")
    {
        cout << "\nВведите № водительского удостоверения >> ";
        drivers_license = check_series_and_number();
    }
    cout << "\nВведите адрес >> ";
    string address = check_address();
    client.FIO = FIO;
    client.passport_details = passport_details;
    client.drivers_license = drivers_license;
    client.address = address;
    return client;
}

// проверка ввода номера автомобиля
string check_number()
{
    string number;
    while (1)
    {
        cin >> number;
        bool flag = true;
        if (number.length() == 9)
        {
            for (int i = 0; i < number.length(); i++)
            {
                if (i == 0 || i == 4 || i == 5)
                    if (number[i] != 'A' && number[i] != 'B' && number[i] != 'E' && number[i] != 'K' && number[i] != 'M' && number[i] != 'H' &&
                        number[i] != 'O' && number[i] != 'P' && number[i] != 'C' && number[i] != 'T' && number[i] != 'Y' && number[i] != 'X')
                        flag = false;
                if ((i >= 1 && i <= 3) || i == 7 || i == 8)
                    if (number[i] < 48 || number[i] > 57)
                        flag = false;
                if (i == 6 && number[i] != 45)
                    flag = false;
            }
        }
        else
            flag = false;
        if (flag) return number;
        cin.clear();
        cin.ignore((numeric_limits<streamsize>::max)(), '\n');
        cout << "Номер автомобиля введен неккоректно. Повторите ввод >> ";
    }
}

// проверка ввода марки автомобиля
string check_brand()
{
    string brand;
    while (1)
    {
        cin >> brand;
        if (brand == "Kia" || brand == "Ford" || brand == "Audi" || brand == "Lada" || brand == "BMW" || brand == "Hyundai" || brand == "Renault")
            return brand;
        else
        {
            cin.clear();
            cin.ignore((numeric_limits<streamsize>::max)(), '\n');
            cout << "Марка автомобиля введена неккоректно. Повторите ввод >> ";
        }
    }
}

// проверка ввода цвета автомобиля
string check_color()
{
    string color;
    while (1)
    {
        cin >> color;
        if (color == "красный" || color == "синий" || color == "серый" || color == "белый" || color == "черный" || color == "желтый")
            return color;
        else
        {
            cin.clear();
            cin.ignore((numeric_limits<streamsize>::max)(), '\n');
            cout << "Цвет автомобиля введен неккоректно. Повторите ввод >> ";
        }
    }
}

// проверка ввода года выпуска автомобиля
int check_year()
{
    int year;
    while (1)
    {
        cin >> year;
        if (cin.fail()) // проверка на ошибку при вводе
        {
            cin.clear(); // возвращаем cin в "обычный" режим работы
            cin.ignore((numeric_limits<streamsize>::max)(), '\n'); // удаление значения предыдущего ввода из входного буфера
            cout << "Некорректный тип данных. Повторите ввод >> ";
        }
        else if (cin.get() != '\n') // проверка на наличие буквы или символа
        {
            cin.ignore((numeric_limits<streamsize>::max)(), '\n'); // удаление значения предыдущего ввода из входного буфера
            cout << "Некорректный тип данных. Повторите ввод >> ";
        }
        else
            if (year >= 1900 && year <= 2021)
                return year;
            else
                cout << "Некорректный год выпуска. Повторите ввод >> ";
    }
}

// проверка ввода признака наличия
bool check_indication_of_availability()
{
    string indication_of_availability;
    cin.ignore(cin.rdbuf()->in_avail());
    while (1)
    {
        getline(cin, indication_of_availability);
        if (indication_of_availability == "есть")
            return true;
        else if (indication_of_availability == "нет")
            return false;
        else
        {
            cin.clear();
            cout << "Признак наличия введен некорректно. Повторите ввод >> ";
        }
    }
}

// функция запрашивает ввести необходимые данные
Car check_car()
{
    Car car;
    cout << "\nВведите номер автомобиля (ANNNAA-NN) >> ";
    car.number = check_number();
    cout << "\nВведите марку автомобиля (Audi, Ford, Kia, Lada, BMW, Hyundai, Renault) >> ";
    car.brand = check_brand();
    cout << "\nВведите цвет автомобиля (красный, синий, серый, белый, черный, желтый) >> ";
    car.color = check_color();
    cout << "\nВведите год выпуска автомобиля (с 1900 по 2021) >> ";
    car.year_of_manufacture = check_year();
    cout << "\nВведите признак наличия в автомобиле ('есть' или 'нет') >> ";
    car.indication_of_availability = check_indication_of_availability();
    return car;
}

// проверка ввода даты
string check_data()
{
    int day;
    int month;
    int year;
    while (1)
    {
        cout << "\nВведите день >> ";
        cin >> day;
        cout << "Введите месяц >> ";
        cin >> month;
        cout << "Введите год >> ";
        cin >> year;
        int Arr[12] = { 31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31 };
        if ((year % 4 == 0 && year % 100 != 0) || year % 400 == 0)
            Arr[1] = 29;
        bool flag = true;
        if (month < 0 || month > 12)
            flag = false;
        if (Arr[month - 1] < day || day < 0)
            flag = false;
        if (year > 2021 || year < 1990)
            flag = false;
        if (flag)
        {
            string data = to_string(day / 10) + to_string(day % 10) + "." + to_string(month / 10) + to_string(month % 10) + "." + to_string(year);
            return data;
        }
        cin.clear();
        cin.ignore((numeric_limits<streamsize>::max)(), '\n');
        cout << "Дата введена некорректно. Повторите ввод >> ";
    }
}
#pragma endregion
/********************************* функции для работы с файлом ***********************************/
#pragma region
// чтение БД о клиентах из файла
void read_BD_of_client(Tree_Node*& p)
{
    fstream file;
    file.open("Client_Base.txt", ios_base::in);
    check_file(file);
    string line;
    while (getline(file, line))
    {
        istringstream iss(line);
        string token;
        int i = 1;
        Client client;
        while (getline(iss, token, ';'))
        {
            if (i == 1) client.FIO = token;
            if (i == 2) client.passport_details = token;
            if (i == 3) client.drivers_license = token;
            if (i == 4) client.address = token;
            i++;
        }
        p = insert_client(p, client);
    }
    file.close();
}

// ф-ция очищает БД о клиентах
void delete_BD_of_client(Tree_Node*& p)
{
    if (!p)
        return;
    delete_BD_of_client(p->right);
    delete_BD_of_client(p->left);
    delete p;
}

// ф-ция сохраняет все изменения БД о клиантах
void save_BD_of_client(Tree_Node* p, fstream& file)
{
    if (!p)
        return;
    else
    {
        file << p->data.FIO << ";" << p->data.passport_details << ";" << p->data.drivers_license << ";" << p->data.address << endl;
        save_BD_of_client(p->left, file);
        save_BD_of_client(p->right, file);
    }
}

// чтение БД об автомобилях из файла
void read_BD_of_car(Car* hash_table)
{
    fstream file;
    file.open("Car_Base.txt", ios_base::in);
    check_file(file);
    for (int i = 0; i < HASH_TABLE_SIZE; i++)
    {
        string line;
        while (getline(file, line))
        {
            istringstream iss(line);
            string token;
            int i = 1;
            Car car;
            while (getline(iss, token, ';'))
            {
                if (i == 1) car.number = token;
                if (i == 2) car.brand = token;
                if (i == 3) car.color = token;
                if (i == 4)
                {
                    int year;
                    stringstream ss(token);
                    ss >> year;
                    car.year_of_manufacture = year;
                }
                if (i == 5)
                {
                    bool indication_of_availability;
                    stringstream ss(token);
                    ss >> indication_of_availability;
                    car.indication_of_availability = indication_of_availability;
                }
                i++;
            }
            push_car(hash_table, car);
        }
    }
    file.close();
}

// ф-ция очищает БД об автомобилях
void delete_BD_of_car(Car* hash_table)
{
    delete[]hash_table;
}

// ф-ция сохраняет все изменения БД об автомобилях
void save_BD_of_car(Car* hash_table, fstream& file)
{
    for (int i = 0; i < HASH_TABLE_SIZE; i++)
    {
        if (hash_table[i].number != "NaN")
            file << hash_table[i].number << ";" << hash_table[i].brand << ";" << hash_table[i].color << ";"
            << hash_table[i].year_of_manufacture << ";" << hash_table[i].indication_of_availability << endl;
    }
}

// чтение БД об аренде автомобилей из файла
void read_BD_of_rent(List_Node*& root)
{
    fstream file;
    file.open("Rent_Base.txt", ios_base::in);
    check_file(file);
    string line;
    while (getline(file, line))
    {
        istringstream iss(line);
        string token;
        int i = 1;
        Rent_a_Car rental_inform;
        while (getline(iss, token, ';'))
        {
            if (i == 1) rental_inform.drivers_license = token;
            if (i == 2) rental_inform.state_registration_number = token;
            if (i == 3) rental_inform.data_of_renting = token;
            if (i == 4) rental_inform.data_of_returning = token;
            i++;
        }
        insert_rental_inform(root, rental_inform);
    }
    file.close();
}

// ф-ция очищает БД об аренде автомобилей
void delete_BD_of_rent(List_Node*& root)
{
    List_Node* temp = root, * next = NULL;
    while (temp != NULL)
    {
        next = temp->next;
        delete temp;
        temp = next;
    }
    delete(next);
    root = NULL;
}

// ф-ция сохраняет все изменения БД об аренде автомобилей
void save_BD_of_rent(List_Node* root, fstream& file)
{
    List_Node* temp = root;
    while (temp != NULL)
    {
        file << temp->rental_inform.drivers_license << ";" << temp->rental_inform.state_registration_number << ";" <<
            temp->rental_inform.data_of_renting << ";" << temp->rental_inform.data_of_returning << ";\n";
        temp = temp->next;
    }
}
#pragma endregion


int main()
{
    SetConsoleCP(1251);
    SetConsoleOutputCP(1251);
    system("color F2");
    Tree_Node* tree_root = NULL; // корень дерева
    read_BD_of_client(tree_root);
    Car* hash_table = new Car[HASH_TABLE_SIZE];
    for (int i = 0; i < HASH_TABLE_SIZE; i++) // инициализация хеш-таблицы
        hash_table[i] = { "NaN", "NaN", "NaN", 0, false };
    read_BD_of_car(hash_table);
    List_Node* list_root = NULL; // корень списка
    read_BD_of_rent(list_root);
    int menu;
    while (1)
    {
        system("Pause");
        system("cls");
        cout << endl << ">> Выберите интересующую БД из меню:\n";
        cout << "   1 - БД о клиентах;\n";
        cout << "   2 - БД об автомобилях;\n";
        cout << "   3 - БД о выдаче на прокат или возврате автомобилей;\n";
        cout << " другое - выход из программы.\n";
        cout << ">> ";
        menu = check_menu();
        switch (menu)
        {
        case 1: // БД O КЛИЕНТАХ
        {
            system("cls");
            cout << endl << ">> Выберите дальнейшее действие из меню:\n";
            cout << "   1 - регистрация нового клиента;\n";
            cout << "   2 - снятие с обслуживания клиента;\n";
            cout << "   3 - просмотр всех зарегистрированных клиентов;\n";
            cout << "   4 - поиск клиента;\n";
            cout << ">> ";
            menu = check_menu();
            switch (menu)
            {
            case 1: // регистрация нового клиента
            {
                system("cls");
                Client new_client = check_client();
                int found = 1; // признак наличия клиента в БД (= 1 - отсутсвует)
                Client client;
                drivers_license_search(tree_root, new_client.drivers_license, "NaN", found, client);
                if (found != 1)
                    cout << "\nКлиент уже есть в БД.";
                else
                    tree_root = insert_client(tree_root, new_client);
                break;
            }
            case 2: // снятие с обслуживания клиента
            {
                system("cls");
                cout << "\nВведите № вод. удостоверения клиента, чтобы снять с обслуживания >> ";
                string data_to_delete = check_series_and_number();
                bool found = false;
                find_client(tree_root, data_to_delete, found);
                if (found)
                    tree_root = delete_client(tree_root, data_to_delete);
                else
                    cout << "Клиент не найден.";
                break;
            }
            case 3: // просмотр всех зарегистрированных клиентов
            {
                system("cls");
                cout << "\n| " << setw(19) << "ФИО" << setw(47) << " |  Паспортные  | Водительское  | " << setw(21) << "Aдрес" << setw(19) << " |\n";
                cout << "| " << setw(42) << " |  данные" << setw(24) << " | удостоверение | " << setw(40) << " |\n";
                cout << "|----------------------------------|--------------|---------------|---------------------------------------|\n";
                reverse_crawl(tree_root);
                break;
            }
            case 4: // поиск клиента
            {
                system("cls");
                cout << endl << ">> Осуществить поиск по:\n";
                cout << "   1 - по № водительского удостоверения;\n";
                cout << "   2 - по фрагментам ФИО или адреса;\n";
                cout << ">> ";
                menu = check_menu();
                switch (menu)
                {
                case 1: // поиск клиента по №
                {
                    system("cls");
                    cout << "\nВведите № водительского удостоверения клиента, которого нужно найти >> ";
                    string drivers_license = check_series_and_number();
                    string number = search_rental_inform_drivers_license(list_root, drivers_license)->rental_inform.state_registration_number;
                    int count_print = 1; // счетчик вывода элементов (чтобы вывело 1 раз)
                    Client client;
                    cout << "\nРезультат поиска:\n\n";
                    cout << "| " << setw(19) << "ФИО" << setw(47) << " |  Паспортные  | Водительское  | " << setw(21) << "Aдрес" << setw(24)\
                        << " | Номер" << setw(8) << " |\n";
                    cout << "| " << setw(42) << " |  данные" << setw(24) << " | удостоверение | " << setw(53) << " | автомобиля |\n";
                    cout << "|----------------------------------|--------------|---------------|---------------------------------------|------------|\n";
                    drivers_license_search(tree_root, drivers_license, number, count_print, client);
                    break;
                }
                case 2: // поиск клиента по ФИО или адреса
                {
                    system("cls");
                    cout << "\nВведите ФИО или адрес клиента, которого нужно найти >> ";
                    string fragment;
                    cin >> fragment;
                    cout << "\nРезультат поиска:\n\n";
                    cout << "| " << setw(22) << "ФИО" << setw(35) << " | Водительское  | " << setw(21) << "Адрес" << setw(19) << " |\n";
                    cout << "| " << setw(57) << " | удостоверение | " << setw(40) << " |\n";
                    cout << "|----------------------------------------|---------------|---------------------------------------|\n";
                    bool found = false;
                    bypassing_clients(tree_root, fragment, found);
                    break;
                }
                }
                break;
            }
            }
            break;
        }
        case 2: // БД ОБ АВТОМОБИЛЯХ
        {
            system("cls");
            cout << endl << ">> Выберите дальнейшее действие из меню:\n";
            cout << "   1 - регистрация нового автомобиля;\n";
            cout << "   2 - удаление сведений об автомобиле;\n";
            cout << "   3 - просмотр всех имеющихся автомобилей;\n";
            cout << "   4 - поиск авомобиля;\n";
            //cout << "   6 - регистрация отправки автомобиля в ремонт;\n";
            //cout << "   7 - регистрация прибытия автомобиля из ремонта;\n";
            cout << ">> ";
            menu = check_menu();
            switch (menu)
            {
            case 1: // регистрация нового автомобиля
            {
                system("cls");
                Car new_car = check_car();
                Car car = find_car(hash_table, new_car.number);
                if (car.number == "NaN")
                    push_car(hash_table, new_car);
                else
                    cout << "\nДанная машина уже есть в БД.\n";
                break;
            }
            case 2: // удаление сведений об автомобиле
            {
                system("cls");
                cout << "Введите номер автомобиля, который нужно удалить >> ";
                string number = check_number();
                delete_car(hash_table, number);
                break;
            }
            case 3: // просмотр всех имеющихся автомобилей
            {
                system("cls");
                cout << "| " << setw(7) << "Номер" << setw(32) << " |  Марка  |  Цвет   | Год  | " << setw(11) << "Наличие" << setw(6) << " |\n";
                cout << "|-----------|---------|---------|------|----------------|\n";
                print_table(hash_table);
                break;
            }
            case 4: // поиск авомобиля
            {
                system("cls");
                cout << endl << ">> Осуществить поиск по:\n";
                cout << "   1 - по гос регистрационному номеру;\n";
                cout << "   2 - по названию марки автомобиля;\n";
                cout << ">> ";
                menu = check_menu();
                switch (menu)
                {
                case 1: // поиск автомобиля по №
                {
                    system("cls");
                    cout << "\nВведите номер автомобиля, который нужно найти >> ";
                    string number = check_number();
                    string drivers_license = search_rental_inform_number(list_root, number)->rental_inform.drivers_license;
                    int found = 1; // признак наличия клиента в БД (= 1 - отсутсвует)
                    Client client;
                    drivers_license_search(tree_root, drivers_license, number, found, client);
                    string FIO = client.FIO;
                    cout << "\nРезультат поиска:\n\n";
                    cout << "| " << setw(7) << "Номер" << setw(31) << " |  Марка  |  Цвет   | Год  |";
                    cout << setw(22) << "ФИО" << setw(35) << " | Водительское  |\n";
                    cout << "|           |         |         |      |                                       | удостоверение |\n";
                    cout << "|-----------|---------|---------|------|---------------------------------------|---------------|\n";
                    Car car = find_car(hash_table, number);
                    cout << "| " << car.number << " | " << car.brand << setw(10 - car.brand.length()) << " | " << car.color << setw(10 - car.color.length()) << " | " << car.year_of_manufacture << " | ";
                    cout << FIO << setw(40 - FIO.length()) << " | ";
                    cout << drivers_license << "  |\n";
                    break;
                }
                case 2: // поиск автомобиля по марке
                {
                    system("cls");
                    cout << "\nВведите марку автомобиля, который нужно найти >> ";
                    string brand = check_brand();
                    cout << "\nРезультат поиска:\n\n";
                    cout << "| " << setw(7) << "Номер" << setw(32) << " |  Марка  |  Цвет   | Год  | " << setw(11) << "Наличие" << setw(6) << " |\n";
                    cout << "|-----------|---------|---------|------|----------------|\n";
                    search_brand(hash_table, brand);
                    break;
                }
                }
                break;
            }
            case 6: // регистрация отправки автомобиля в ремонт
            {
                system("cls");

                break;
            }
            case 7: // регистрация прибытия автомобиля из ремонта
            {
                system("cls");

                break;
            }
            }
            break;
        }
        case 3: // БД ОБ АРЕНДЕ АВТОМОБИЛЕЙ
        {
            system("cls");
            cout << endl << ">> Выберите дальнейшее действие из меню:\n";
            cout << "   1 - регистрация выдачи клиенту автомобиля на прокат;\n";
            cout << "   2 - регистрация возврата автомобиля от клиентов;\n";
            cout << "   3 - вывод списка арендованных автомобилей и данных об аренде;\n";
            cout << ">> ";
            menu = check_menu();
            switch (menu)
            {
            case 1: // выдача
            {
                system("cls");
                // собираем данные об аренде
                Rent_a_Car rental_inform;
                print_free_car(hash_table);
                cout << "\nВыберите свободный автомобиль >> ";
                rental_inform.state_registration_number = check_number();
                cout << "\nВведите № водительского удостоверения >> ";
                string drivers_license = check_series_and_number();
                // проверяем на наличие этого клиента в БД клиента, иначе - добавляем
                int found = 1; // признак наличия клиента в БД (= 1 - отсутсвует)
                Client client;
                drivers_license_search(tree_root, drivers_license, rental_inform.state_registration_number, found, client);
                if (found != 1)
                {
                    rental_inform.drivers_license = drivers_license;
                    break;
                }
                else
                {
                    cout << "\nСледует добавить нового клиента:";
                    Client new_client = check_client(drivers_license);
                    insert_client(tree_root, new_client);
                    rental_inform.drivers_license = drivers_license;
                }
                cout << "\nУкажите дату выдачи:\n";
                rental_inform.data_of_renting = check_data();
                cout << "\nУкажите дату возврата:\n";
                rental_inform.data_of_returning = check_data();
                // добавляем запись в список
                insert_rental_inform(list_root, rental_inform);
                // помечаем занятость автомобиля
                find_car(hash_table, rental_inform.state_registration_number, 1);
                // сортируем список
                list_root = merge_sort(list_root);
                break;
            }
            case 2: // возврат
            {
                system("cls");
                delete_rental_inform(tree_root, hash_table, list_root);
                break;
            }
            case 3: // вывод списка арендованных автобомилей и данных об аренде
            {
                system("cls");
                cout << "\nСписок арендованных автомобилей и данных об аренде:\n\n";
                cout << "| Водительское  |   Номер   |   Дата     |  Дата      |\n";
                cout << "| удостоверение |   авто    |   выдачи   |  возврата  |\n";
                cout << "|---------------|-----------|------------|------------|\n";
                print_rental_inform(list_root);
                break;
            }
            }
            break;
        }
        default:
        {
            system("cls");
            // работа с БД о клиентах
            fstream file_for_client;
            file_for_client.open("Client_Base.txt", ios_base::out | ios_base::trunc);
            fstream file_for_car;
            check_file(file_for_client);
            save_BD_of_client(tree_root, file_for_client);
            delete_BD_of_client(tree_root);
            // работа с БД об автомобилях
            file_for_client.close();
            file_for_car.open("Car_Base.txt", ios_base::out | ios_base::trunc);
            check_file(file_for_car);
            save_BD_of_car(hash_table, file_for_car);
            delete_BD_of_car(hash_table);
            file_for_car.close();
            // работа с БД об аренде автомобилей
            fstream file_for_rent;
            file_for_rent.open("Rent_Base.txt", ios_base::out | ios_base::trunc);
            check_file(file_for_rent);
            save_BD_of_rent(list_root, file_for_rent);
            delete_BD_of_rent(list_root); delete (list_root);
            file_for_rent.close();
            check_leak();
            return 0;
        }
        }
    }
}

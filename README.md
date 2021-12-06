# PRiR_LAB7_1

Zadanie nr 1 z laboratoriów 7 z przedmiotu PRiR. Implementacja symulatora autobusu z wykorzystaniem mpi. Tak jak w poleceniu symulator jest wzorowany symulatorem lotniska z wykładu 6. 
Autobusy posiadają zajezdnię do której mogą zjechać i zatankować jeśli są wolne stanowiska. Autobusy mogą również zjeżdzać na przystanki. 
Jeśli paliwo w autobusie się skończy autobus zostaje porzucony. Dodatkowymi rozszeżeniami projektu jest licznik pasażerów w autobusie. 
Liczba pasażerów wsiadających i wysiadających jest losowana na przystankach. Dodatkowo każdy autobus posiada 5% szany na to że się zepsuje wtedy również zostaje porzucony. 
Program trwa dopóki wszystkie autobusy nie zostaną porzucone.

OPIS KODU:

Import bibliotek

    #include <stdio.h>
    #include <stdlib.h>
    #include <unistd.h>
    #include <time.h>
    #include "mpi.h"
    
Zdefiniowanie stałych 

    #define REZERWA 400
    #define PRZYSTANEK 1
    #define START 2
    #define JAZDA 3
    #define ZJAZD_DO_ZAJEZDNI 4
    #define KOLIZJA 5
    #define TANKUJ 4000
    
Inicjalizacja zmiennych 

    int liczba_pasarzerow = 0;
    int paliwo = 4000;
    int ZATRZYMAJ = 1, NIE_ZATRZYMUJ = 0;
    int liczba_procesow;
    int nr_procesu,ilosc_autobusow,ilosc_miejs_w_zajezdni = 4,ilosc_zajetych_miejsc_w_zajezdni = 0,tag=1;
    wyslij[2];
    int odbierz[2];
    MPI_Status mpi_status;
    
funkcja wysyłająca dane

    void Wyslij(int nr_autobusu, int stan) 
    {
      wyslij[0] = nr_autobusu;
      wyslij[1] = stan;
      MPI_Send(&wyslij, 2, MPI_INT, 0, tag, MPI_COMM_WORLD);
      sleep(1);
    }
    
 funkcja definiująca działanie zajezdni. Inicjalizuje ona zmienne wyświetla komunikaty i następnie na podstawie odpowiednich statusów wyświetla odpowiednie komunikaty i dokonuje operacje przypisane do danego statusu np. zmniejsza liczbę wolnych pasów albo zajmuje wolny pas i wysyła dane
 
     void Zajezdnia(int liczba_procesow){
      int nr_autobusu, status;
      ilosc_autobusow = liczba_procesow -1;
      printf("Kierowcy są gotowi do jazdy \n \n \n");
      printf("W zajezdni jest %d stanowisk\n", ilosc_miejs_w_zajezdni);
      sleep(2);
      while(ilosc_miejs_w_zajezdni <= ilosc_autobusow){
        MPI_Recv(&odbierz,2,MPI_INT,MPI_ANY_SOURCE,tag,MPI_COMM_WORLD, &mpi_status);
        nr_autobusu = odbierz[0];
        status = odbierz[1];
        if(status == 1)	printf("Autobus %d stoi w na przystanku\n", nr_autobusu);

        if(status==2){
          printf("Autobus %d wyjezdza z zajezdni zwalniajac pas nr %d\n", nr_autobusu, ilosc_zajetych_pasow);
          ilosc_zajetych_pasow--;
        }
        if(status == 3)	printf("Autobus %d jest w trasie\n", nr_autobusu);

        if(status == 4){
          if(ilosc_zajetych_pasow<ilosc_miejs_w_zajezdni){
            printf("Autobus %d wszyscy pasarzerowie wysiadaja\n", nr_autobusu);
            ilosc_zajetych_pasow++;
            MPI_Send(&ZATRZYMAJ, 1, MPI_INT, nr_autobusu, tag, MPI_COMM_WORLD);
          }
          else{
            printf("Autobus %d wszyscy pasarzerowie wysiadaja\n", nr_autobusu);
            MPI_Send(&NIE_ZATRZYMUJ, 1, MPI_INT, nr_autobusu, tag, MPI_COMM_WORLD);
          }
        }
        if(status == 5){
          ilosc_autobusow--;
          printf("Ilosc autobusow %d\n", ilosc_autobusow);
        }
      }
      printf("Program zakonczyl dzialanie\n");
    }
    
 funkcja autobus służy do działania pojedyńczego autobusu. inicjalizuje zmienne a następnie w pętli losuję wartość i jeśli równa jest 6 to autobus się psuje. W przeciwnym razie 
 sprawdza stany i dokonuje odpowiednich działań typu wysyłania i odbierania informacji a także wyświetla odpowiednie komunikaty 
 
     void Autobus(){
      int stan,suma,i;
      stan = JAZDA;
      while(1){
        if(rand()%20 == 6) {
          stan=KOLIZJA;
          printf("Autobus ulegl usterce zostaje porzucony\n");
          Wyslij(nr_procesu,stan);
          return;
        }
        if(stan == 1){
          if(rand()%2 == 1){
            stan = START;
            paliwo = TANKUJ;
            printf("Sprawdzam gotowosc do wyjazdu z przystanku nr %d\n",nr_procesu);
            liczba_pasarzerow -= rand()%5;
            printf("Liczba pasarzerow $d\n",liczba_pasarzerow);
            if(liczba_pasarzerow < 0) liczba_pasarzerow = 0;
            Wyslij(nr_procesu,stan);
          }
          else	Wyslij(nr_procesu,stan);
        }
        else if(stan == 2){
          stan=JAZDA;
          printf("Wyjezdzam z zajezdni autobus nr %d\n",nr_procesu);
          liczba_pasarzerow += rand()%5;
          if(liczba_pasarzerow > 100) liczba_pasarzerow = 100;
          printf("Liczba pasarzerow $d\n",liczba_pasarzerow);
          Wyslij(nr_procesu,stan);
        }
        else if(stan == 3){
          paliwo -= rand()%400;
          if(paliwo <= REZERWA){
            stan=ZJAZD_DO_ZAJEZDNI;
            printf("Potrzebuje zjechać do zajezdni\n");
            Wyslij(nr_procesu,stan);
          }
          else for(i = 0; rand()%10000; i++);
        }
        else if(stan == 4){
          int temp;
          MPI_Recv(&temp, 1, MPI_INT, 0, tag, MPI_COMM_WORLD, &mpi_status);
          if(temp == ZATRZYMAJ){
            stan=PRZYSTANEK;
            printf("Zjezdzam do zajezdni, autobus nr %d\n", nr_procesu);
            liczba_pasarzerow = 0;
          }
          else {
            paliwo -= rand()%400;
            if(paliwo > 0)	Wyslij(nr_procesu,stan);
            else{
              stan = KOLIZJA;
              printf("Koniec paliwa autobus zostaje porzucony\n");
              Wyslij(nr_procesu,stan);
              return;
            }
          }
        }
      }
    }
    
funkcja main wywołuje inicjalizuje zienne dla mpi i wywołuje sprawdza czy numer proesu jest równy 0 jeśli tak to wywołuje funkcję zajezdnia w przeciwnym wypadku wywołuje funkcję autobus dla danego procesu.

    int main(int argc, char *argv[])
    {
      MPI_Init(&argc, &argv);
      MPI_Comm_rank(MPI_COMM_WORLD,&nr_procesu);
      MPI_Comm_size(MPI_COMM_WORLD,&liczba_procesow);

      srand(time(NULL));

      if(nr_procesu == 0) Zajezdnia(liczba_procesow);
      else Autobus();

      MPI_Finalize();
      return 0;
    }

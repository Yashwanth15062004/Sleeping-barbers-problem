# Sleeping-barbers-problem
Problem statement:
The Sleeping Barber problem is a classic synchronization problem that involves multiple barbers and multiple customers. The problem assumes a barbershop with a certain number of chairs for customers to wait in, and a certain number of barbers who can cut hair at the same time. Customers arrive randomly and either sit in a chair if one is available or leave if all chairs are occupied. If a barber is available, they begin cutting the hair of a customer. If all barbers are busy, customers must wait in their chairs until a barber becomes available.

The challenge in this problem is to synchronize the access to shared resources (chairs and barbers) and to prevent race conditions where two customers might try to sit in the same chair, or two barbers might try to cut the hair of the same customer. Additionally, the solution must ensure that the barbers only cut hair when there is a customer in the chair.

Solution:


* Input
number of barbers = n_brbs
number of customers=n_cust
number of waiting seats=wtsts

*semaphore initialisation and usage
1. For mutual exclusion between waiting seats of entering customers we have used m_wtsts binary semaphore initialized to 1.
2.For mutual exclusion between barbers we used m_brbs initialized to 1.
3.For mutual exclusion between the un-served customers(customers who exited the shop with out the job has done) we used m_unser initialised to 1.
4.To keep track of the number of ready barbers we used rdb(counting sem) initialized to 0.
5.brb[i] semaphore is used such that whenever customer signals it then barber starts doing haircut. It is initialized to "0" sothat barbers are blocked initially.

Initially, all the barber threads are created. Each thread is given an ID number and calls the function "barber_f". is_slp[ID] = 1 is set for all the barber ID's such that initially every barber is sleeping,also semaphore(rdb) is signaled.
Now,when the Customer threads are created, each thread checks the no.of empty seats. If the empty seats are "0" then the customer leaves the shop by incrementing unserved customers,m_unser sem is used for mutual exclusion between unserved customers. Else he will enter the shop and sit and check for the ready barbers(barbers who are sleeping). If there are ready barbers  then the customer need to select the first ready barber (which is implemented using a for loop) and signal the brb[i] semaphore so that the particular barber cut the hair of the customer who woke him up.
Finally, we print the corresponding ID numbers of the barber and the customer where the haircut is done.

```c
#include <stdio.h>
#include <pthread.h>
#include <time.h>
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>
#define n_barb 4  //No.of barbers
#define n_cust 20 //No.of customers
#define wtsts 10  //No.of waiting seats

typedef struct {
    int count;
    pthread_mutex_t lock;
    pthread_cond_t non_zero;
} semaphore_t;

pthread_mutex_t mutex;

void s_initialize(semaphore_t *s, int count) {
    s->count = count;
    pthread_mutex_init(&s->lock, NULL);
    pthread_cond_init(&s->non_zero, NULL);
}

void s_wait(semaphore_t *s) {
    pthread_mutex_lock(&s->lock);
    while (s->count == 0) {
        pthread_cond_wait(&s->non_zero, &s->lock);
    }
    s->count--;
    pthread_mutex_unlock(&s->lock);
}

void s_signal(semaphore_t *s) {
    pthread_mutex_lock(&s->lock);
    s->count++;
    pthread_cond_signal(&s->non_zero);
    pthread_mutex_unlock(&s->lock);
}

void s_destroy(semaphore_t *s) {
    pthread_mutex_destroy(&s->lock);
    pthread_cond_destroy(&s->non_zero);
}

semaphore_t brb[n_barb];
int is_slp[n_barb];
semaphore_t m_wtsts;
semaphore_t m_brbs;
semaphore_t rdb;
semaphore_t m_unser;
int slping_bs;
int emp_wts=wtsts;
int unserv=0;
void * barber_f(void *bn){
    int ID=(int)bn;
    while(1){
        s_wait(&m_brbs);
         is_slp[ID]=1;
         slping_bs++;
        s_signal(&rdb);
        s_signal(&m_brbs);
        s_wait(&brb[ID]);  
        //cut hair of the customer who woke him up
        sleep(3); //haircut takes some time
    }

}
void * customer_f(void *cn){
    int ID=(int)cn;
    s_wait(&m_wtsts); // entering customers synchronisation
     if(emp_wts==0){
         s_signal(&m_wtsts); 
            //leave the shop/
        s_wait(&m_unser);
         unserv++;  // synro btm unserved cust
        s_signal(&m_unser); 
     }
     else{
        emp_wts--;
        s_signal(&m_wtsts); 


        s_wait(&rdb);
         s_wait(&m_wtsts); 
          emp_wts++;
         s_signal(&m_wtsts);
         s_wait(&m_brbs); //mutual exclusion btw barbers
          int i=0;
           for(i=0;i<n_barb;i++){
            if(is_slp[i]==1){
                //wake him up
                is_slp[i]=0;   
                slping_bs--;
                s_signal(&brb[i]);  
                break;

             }
           }
         s_signal(&m_brbs);
         printf("customer %d is getting haircut from barber %d\n",ID,i);
     }
}

int main(){
    s_initialize(&m_wtsts,1);
    s_initialize(&m_brbs,1);
    s_initialize(&m_unser,1);
    s_initialize(&rdb,0);
    for(int i=0;i<n_barb;i++){
        s_initialize(&brb[i],0);
    }
    pthread_t barber[n_barb];
    pthread_t customer[n_cust];
    for(int i=0;i<n_barb;i++){
        pthread_create(&barber[i],NULL,&barber_f,(void*)i);
    }
    for(int i=0;i<n_cust;i++){
         sleep(0.01);
        pthread_create(&customer[i],NULL,&customer_f,(void*)i);
    }
    
    for(int i=0;i<n_cust;i++){
        pthread_join(customer[i],NULL);
    }
     
    printf("No.of Unserverd customers : %d\n",unserv);

    pthread_mutex_destroy(&mutex);
    s_destroy(&brb[n_barb]);
    s_destroy(&m_wtsts);
    s_destroy(&m_brbs);
    s_destroy(&rdb);
    s_destroy(&m_unser);

    return 0;
}
```


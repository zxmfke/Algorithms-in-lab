<h1>FourBit Trie</h1>

<h4>方法介绍:</h4>
将原来binary trie每次读prefix的1 bit改成一次读4bits的FourBit Trie，用来加快IP lookup的速度

<h4>示意图:</h4>

![image](https://github.com/zxmfke/Algorithms-in-lab/blob/master/img_folder./Fourbit_Trie_Structure.jpg)

<h4>程式:</h4>

```c
#include<stdlib.h>
#include<stdio.h>
#include<string.h>
////////////////////////////////////////////////////////////////////////////////////
struct ENTRY{
	unsigned int ip;
	unsigned char len;
	unsigned char port;
};
////////////////////////////////////////////////////////////////////////////////////
inline unsigned long long int rdtsc()
{
	unsigned long long int x;
	asm   volatile ("rdtsc" : "=A" (x));
	return x;
}
////////////////////////////////////////////////////////////////////////////////////
struct list{//structure of 4-bit trie
	unsigned int port;
	unsigned char len;
	struct list *ptrlist[16];
};
typedef struct list node;
typedef node *btrie;
////////////////////////////////////////////////////////////////////////////////////
/*global variables*/
btrie root;
unsigned int *query;
int num_entry=0;
int num_query=0;
struct ENTRY *table;
int N=0;//number of nodes
unsigned long long int begin,end,total=0;
unsigned long long int *clock;
int num_node=0;//total number of nodes in the binary trie
////////////////////////////////////////////////////////////////////////////////////
btrie create_node(){
	btrie temp;
	num_node++;
	temp=(btrie)malloc(sizeof(node));
	int i;
	for(i = 0; i < 16; i++) temp -> ptrlist[i] = NULL;
	temp->port=256;//default port
	temp -> len = 0;
	return temp;
}
////////////////////////////////////////////////////////////////////////////////////
void add_node(unsigned int ip,unsigned char len,unsigned char nexthop){
	btrie ptr=root;
	int i;
	unsigned int temp;
	for(i=3;i<len;i+=4){
		temp = (ip >> (31 - i)) & 15;
		if(ptr -> ptrlist[temp] == NULL){
			ptr -> ptrlist[temp] = create_node();
		}
		ptr = ptr -> ptrlist[temp];
		if(i == len - 1 && len > ptr -> len){
			ptr -> port = nexthop;
			ptr -> len = len;
		}
	}
	if(len % 4 != 0){
		unsigned int max;
		unsigned int bitmask[3] = {8, 12, 14};
		temp = (ip >> (31 - i)) & bitmask[len % 4 - 1];
		max = temp + (1 << (4 - (len % 4)));
		for(; temp < max; temp++){
			if(ptr -> ptrlist[temp] == NULL)
				ptr -> ptrlist[temp] = create_node();
			//ptr = ptr -> ptrlist[temp];
			if(len > ptr -> ptrlist[temp] -> len){
				ptr -> ptrlist[temp] -> port = nexthop;
				ptr -> ptrlist[temp] -> len = len;
			}
		}
	}
}
////////////////////////////////////////////////////////////////////////////////////
void read_table(char *str,unsigned int *ip,int *len,unsigned int *nexthop){
	char tok[]="./";
	char buf[100],*str1; 
	unsigned int n[4];
	sprintf(buf,"%s\0",strtok(str,tok));
	n[0]=atoi(buf);
	sprintf(buf,"%s\0",strtok(NULL,tok));
	n[1]=atoi(buf);
	sprintf(buf,"%s\0",strtok(NULL,tok));
	n[2]=atoi(buf);
	sprintf(buf,"%s\0",strtok(NULL,tok));
	n[3]=atoi(buf);
	*nexthop=n[2];
	str1=(char *)strtok(NULL,tok);
	if(str1!=NULL){
		sprintf(buf,"%s\0",str1);
		*len=atoi(buf);
	}
	else{
		if(n[1]==0&&n[2]==0&&n[3]==0)
			*len=8;
		else
			if(n[2]==0&&n[3]==0)
				*len=16;
			else
				if(n[3]==0)
					*len=24;
	}
	*ip=n[0];
	*ip<<=8;
	*ip+=n[1];
	*ip<<=8;
	*ip+=n[2];
	*ip<<=8;
	*ip+=n[3];
}
////////////////////////////////////////////////////////////////////////////////////
void search(unsigned int ip){
	int j;
	btrie current=root,temp=NULL;
	/*FILE *fp;
	fp = fopen("output.txt", "a");*/
	for(j = 3; j < 32; j += 4){
		if(current -> ptrlist[(ip >> (31 - j)) & 15] != NULL){
			current = current -> ptrlist[(ip >> (31 - j)) & 15];
		}
		else{
			break;
		}
		if(current->port!=256){
			temp=current;
		}
	}
	/*if(temp==NULL)
	  fprintf(fp, "default\n");
	else
	  fprintf(fp, "%u\n", temp->port);
	fclose(fp);*/
}
////////////////////////////////////////////////////////////////////////////////////
void set_table(char *file_name){
	FILE *fp;
	int len;
	char string[100];
	unsigned int ip,nexthop;
	fp=fopen(file_name,"r");
	while(fgets(string,50,fp)!=NULL){
		//read_table(string,&ip,&len,&nexthop);
		num_entry++;
	}
	rewind(fp);
	table=(struct ENTRY *)malloc(num_entry*sizeof(struct ENTRY));
	num_entry=0;
	while(fgets(string,50,fp)!=NULL){
		read_table(string,&ip,&len,&nexthop);
		table[num_entry].ip=ip;
		table[num_entry].port=nexthop;
		table[num_entry++].len=len;
	}
}
////////////////////////////////////////////////////////////////////////////////////
void set_query(char *file_name){
	FILE *fp;
	int len;
	char string[100];
	unsigned int ip,nexthop;
	fp=fopen(file_name,"r");
	while(fgets(string,50,fp)!=NULL){
		//read_table(string,&ip,&len,&nexthop);
		num_query++;
	}
	rewind(fp);
	query=(unsigned int *)malloc(num_query*sizeof(unsigned int));
	clock=(unsigned long long int *)malloc(num_query*sizeof(unsigned long long int));
	num_query=0;
	while(fgets(string,50,fp)!=NULL){
		read_table(string,&ip,&len,&nexthop);
		query[num_query]=ip;
		clock[num_query++]=10000000;
	}
}
////////////////////////////////////////////////////////////////////////////////////
void create(){
	int i;
	root=create_node();
	begin=rdtsc();
	for(i=0;i<num_entry;i++){
		add_node(table[i].ip,table[i].len,table[i].port);
	}
	end=rdtsc();
}
////////////////////////////////////////////////////////////////////////////////////
void delete(){
	btrie ptr=root;
	int i, j;
	unsigned int temp, ip;
	unsigned char len, nexthop;
	begin=rdtsc();
	for(j = 0; j < num_query; j++){
		ip = table[j].ip;
		len = table[j].len;
		nexthop = table[j].port;
		ptr = root;
		for(i=3;i<len;i+=4){
			temp = (ip >> (31 - i)) & 15;
			ptr = ptr -> ptrlist[temp];
			if(i == len - 1 && ptr -> port == nexthop){
				ptr -> port = 256;
			}
		}
		if(len % 4 != 0){
			unsigned int max;
			unsigned int bitmask[3] = {8, 12, 14};
			temp = (ip >> (31 - i)) & bitmask[len % 4 - 1];
			max = temp + (1 << (4 - (len % 4)));
			for(; temp < max; temp++){
				if(ptr -> ptrlist[temp] -> port == nexthop)
					ptr -> ptrlist[temp] -> port = 256;
			}
		}
	}
	end=rdtsc();
}
////////////////////////////////////////////////////////////////////////////////////
void count_node(btrie r){
	if(r==NULL)
		return;
	int i;
	for(i = 0; i < 16; i++)
		count_node(r -> ptrlist[i]);
}
////////////////////////////////////////////////////////////////////////////////////
void CountClock()
{
	unsigned int i;
	unsigned int* NumCntClock = (unsigned int* )malloc(50 * sizeof(unsigned int ));
	for(i = 0; i < 50; i++) NumCntClock[i] = 0;
	unsigned long long MinClock = 10000000, MaxClock = 0;
	for(i = 0; i < num_query; i++)
	{
		if(clock[i] > MaxClock) MaxClock = clock[i];
		if(clock[i] < MinClock) MinClock = clock[i];
		if(clock[i] / 100 < 50) NumCntClock[clock[i] / 100]++;
		else NumCntClock[49]++;
	}
	printf("(MaxClock, MinClock) = (%5llu, %5llu)\n", MaxClock, MinClock);
	
	for(i = 0; i < 50; i++)
	{
		printf("%d : %d\n", (i + 1) * 100, NumCntClock[i]);
	}
	return;
}
////////////////////////////////////////////////////////////////////////////////////
void reset(){
	int i, j;
	unsigned int temp;

	for(i = 0; i < 407218; i++){
		j = rand() % 407218;
		temp = query[i];
		query[i] = query[j];
		query[j] = temp;
	}
}
////////////////////////////////////////////////////////////////////////////////////
int main(int argc,char *argv[]){
	/*if(argc!=3){
		printf("Please execute the file as the following way:\n");
		printf("%s  routing_table_file_name  query_table_file_name\n",argv[0]);
		exit(1);
	}*/
	int i,j;
	//set_query(argv[2]);
	//set_table(argv[1]);
	set_query("IPv4-Prefix-AS6447-2012-02-07-1-407218.txt");
	set_table("IPv4-Prefix-AS6447-2012-02-07-1-407218.txt");
	create();
	//reset();
	printf("Avg. Inseart: %llu\n",(end-begin)/num_entry);
	printf("number of nodes: %d\n",num_node);
	printf("Total memory requirement: %d KB\n",((num_node*72)/1024));
	////////////////////////////////////////////////////////////////////////////
	for(j=0;j<100;j++){
		for(i=0;i<num_query;i++){
			begin=rdtsc();
			search(query[i]);
			end=rdtsc();
			if(clock[i]>(end-begin))
				clock[i]=(end-begin);
		}
	}
	total=0;
	for(j=0;j<num_query;j++)
		total+=clock[j];
	printf("Avg. Search: %llu\n",total/num_query);
	CountClock();
	delete();
	printf("Avg. Delete: %llu\n",(end-begin)/num_entry);
	////////////////////////////////////////////////////////////////////////////
	//count_node(root);
	//printf("There are %d nodes in binary trie\n",N);
	return 0;
}
```

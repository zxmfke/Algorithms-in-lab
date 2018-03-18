<h1>Tree Bitmap</h1>

<h4>方法描述:</h4>
Tree Bitmap是个可以快速搜寻，快速更新的multibit trie algorithm。Tree Bitmap是一个3-bit stride的概念，如图1，从Binary trie中的root开始，将三层的节点存为Tree Bitmap中的一个新节点。Tree Bitmap演算法中，如图2，每个节点中会有两个bitmap阵列，其一是internal prefixes array，来判断该node是否为longest prefixes，其二是external prefixes array，一个指标阵列，判断最后一层的节点是否有子节点。还有两个阵列，分別是储存最后一层的子节点，和nexthop的array。该演算法，通过node中的子节点指标阵列，将一个Tree Bitmap node的子节点全都存到这个阵列内，从而减少node的数量。

<h4>示意图:</h4>

图1
![image](https://github.com/zxmfke/Algorithms-in-lab/blob/master/img_folder./Tree_Bitmap_structure.jpg)

图2
![image](https://github.com/zxmfke/Algorithms-in-lab/blob/master/img_folder./Tree_Bitmap_node_structure.jpg.png)

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
struct list{//structure of binary trie
	unsigned int port;
	struct list *left,*right;
};
typedef struct list node;
typedef node *btrie;

struct link{//structure of super node trie
	unsigned char *internal;
	unsigned char *external;
	unsigned int *nexthop;
	struct link *pt[16];
};
typedef struct link linker;
typedef linker *supertrie;
////////////////////////////////////////////////////////////////////////////////////
/*global variables*/
btrie root;
supertrie superoot;
unsigned int *query;
int num_entry=0;
int num_query=0;
struct ENTRY *table;
int N=0;//number of nodes
unsigned long long int begin,end,total=0;
unsigned long long int *clock;
int num_node=0;//total number of nodes in the binary trie
int k,j;
int super_node=0;

////////////////////////////////////////////////////////////////////////////////////
btrie create_node(){
	btrie temp;
	num_node++;
	temp=(btrie)malloc(sizeof(node));
	temp->right=NULL;
	temp->left=NULL;
	temp->port=256;//default port
	return temp;
}
////////////////////////////////////////////////////////////////////////////////////
void add_node(unsigned int ip,unsigned char len,unsigned char nexthop){
	btrie ptr=root;
	int i;
	for(i=0;i<len;i++){
		if(ip&(1<<(31-i))){
			if(ptr->right==NULL)
				ptr->right=create_node(); // Create Node
			ptr=ptr->right;
			if((i==len-1)&&(ptr->port==256))
				ptr->port=nexthop;
		}
		else{
			if(ptr->left==NULL)
				ptr->left=create_node();
			ptr=ptr->left;
			if((i==len-1)&&(ptr->port==256))
				ptr->port=nexthop;
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
void search(unsigned int ip,FILE *f){
	int j;
	btrie current=root,temp=NULL;
	for(j=31;j>=(-1);j--){
		if(current==NULL)
			break;
		if(current->port!=256)
			temp=current;
		if(ip&(1<<j)){
			current=current->right;
		}
		else{
			current=current->left; 
		}
	}
	  if(temp==NULL)
	  fprintf(f,"default\n");
	  else
	  fprintf(f,"%u\n",temp->port);
}
////////////////////////////////////////////////////////////////////////////////////
supertrie bitbuild(btrie poi){
	int i;
	supertrie t;
	t = (supertrie) malloc(sizeof(linker));
	//////initialize
	t->internal = NULL;
	t->external = NULL;
	t->nexthop = NULL;
	for(i=0;i<16;i++){
	t->pt[i] = NULL;
	}
	//////
	btrie ptr = poi;
	btrie *temp;
	temp = (btrie *) malloc (16*sizeof(node));
	unsigned int *nexthop;
	nexthop = (unsigned int*) malloc(16*sizeof(unsigned int));
	unsigned char *internal;
	internal = (unsigned char*) malloc(16*sizeof(unsigned char));
	unsigned char *external;
	external = (unsigned char*) malloc(16*sizeof(unsigned char));
	for(i=0;i<16;i++){
		temp[i] = NULL;
		nexthop[i]=256;
		internal[i]=0;
		external[i]=0;
	}
	////////////level-0
	temp[0] = ptr;///root
	if(ptr->port!=256) {
	nexthop[0] = ptr->port;
	internal[0] = 1;
	}
	if(ptr->left!=NULL){
		ptr = ptr->left;
		if(ptr->port!=256){
		nexthop[1] = ptr->port;
		internal[1] = 1;
		}
		temp[1] = ptr;
		ptr = temp[0];
	}
	if(ptr->right!=NULL){
		ptr = ptr->right;
		if(ptr->port!=256){
		nexthop[2] = ptr->port;
		internal[2] = 1;
		}
		temp[2] = ptr;
		ptr = temp[0];
	}
	///////////level-1
	for(j=1 ; j<3 ; j++){
		ptr = temp[j];
		if(ptr==NULL) continue;
		if(ptr->left!=NULL){
			ptr = ptr->left;
			if(ptr->port!=256){
			nexthop[2*j+1] = ptr->port;
			internal[2*j+1] = 1;
			}
			temp[2*j+1] = ptr;
			ptr = temp[j];
		}
		
		if(ptr->right!=NULL){
			ptr = ptr->right;
			if(ptr->port!=256){
			nexthop[2*j+2] = ptr->port;
			internal[2*j+2] = 1;
			}
			temp[2*j+2] = ptr;
			ptr = temp[j];
		}	
	}
	///level2
	for(j=3 ; j<7 ; j++){
		ptr = temp[j];
		if(ptr==NULL) continue;
		if(ptr->left!=NULL){
			ptr = ptr->left;
			if(ptr->port!=256){
			nexthop[2*j+1] = ptr->port;
			internal[2*j+1] = 1;
			}
			temp[2*j+1] = ptr;
			ptr = temp[j];
		}
		if(ptr->right!=NULL){
			ptr = ptr->right;
			if(ptr->port!=256){
			nexthop[2*j+2] = ptr->port;
			internal[2*j+2] = 1;
			}
			temp[2*j+2] = ptr;
			ptr = temp[j];
		}
	}
	t->nexthop = nexthop; 
	t->internal = internal;
	///level3  next supernode
	//node 7
	if(temp[7]!=NULL){
		ptr = temp[7];
		if(ptr->left!=NULL){
			ptr = ptr->left;
			external[0] = 1;		
			t->pt[0] = bitbuild(ptr);
			ptr = temp[7];
		}
		if(ptr->right!=NULL){
			ptr = ptr->right;
			external[1] = 1;		
			t->pt[1] = bitbuild(ptr);
		}
	}
	//node 8
	if(temp[8]!=NULL){
		ptr = temp[8];
		if(ptr->left!=NULL){
			ptr = ptr->left;
			external[2] = 1;
			t->pt[2] = bitbuild(ptr);
			ptr = temp[8];
		}
		if(ptr->right!=NULL){
			ptr = ptr->right;
			external[3] = 1;
			t->pt[3] = bitbuild(ptr);
		}
	}
	//node 9
	if(temp[9]!=NULL){
		ptr = temp[9];
		if(ptr->left!=NULL){
			ptr = ptr->left;
			external[4] = 1;
			t->pt[4] = bitbuild(ptr);
			ptr = temp[9];
		}
		if(ptr->right!=NULL){
			ptr = ptr->right;
			external[5] = 1;
			t->pt[5] = bitbuild(ptr);
		}
	}
	//node 10
	if(temp[10]!=NULL){
		ptr = temp[10];
		if(ptr->left!=NULL){
			ptr = ptr->left;
			external[6] = 1;
			t->pt[6] = bitbuild(ptr);
			ptr = temp[10];
		}
		if(ptr->right!=NULL){
			ptr = ptr->right;
			external[7] = 1;
			t->pt[7] = bitbuild(ptr);
		}
	}
	//node11
	if(temp[11]!=NULL){
		ptr = temp[11];
		if(ptr->left!=NULL){
			ptr = ptr->left;
			external[8]=1;
			t->pt[8] = bitbuild(ptr);
			ptr = temp[11];
		}
		if(ptr->right!=NULL){
			ptr = ptr->right;
			external[9]=1;
			t->pt[9] = bitbuild(ptr);
		}
	}
	//node12	
	if(temp[12]!=NULL){
	ptr = temp[12];
		if(ptr->left!=NULL){
			ptr = ptr->left;
			external[10]=1;
			t->pt[10] = bitbuild(ptr);
			ptr = temp[12];
		}
		if(ptr->right!=NULL){
			ptr = ptr->right;
			external[11]=1;
			t->pt[11] = bitbuild(ptr);
		}
	}
	//node13	
	if(temp[13]!=NULL){
	ptr = temp[13];
		if(ptr->left!=NULL){
			ptr = ptr->left;
			external[12]=1;
			t->pt[12] = bitbuild(ptr);
			ptr = temp[13];
		}
		if(ptr->right!=NULL){
			ptr = ptr->right;
			external[13]=1;
			t->pt[13] = bitbuild(ptr);
		}
	}
	//node14
	if(temp[14]!=NULL){
		ptr = temp[14];
		if(ptr->left!=NULL){
			ptr = ptr->left;
			external[14]=1;
			t->pt[14] = bitbuild(ptr);
			ptr = temp[14];
		}
		if(ptr->right!=NULL){
			ptr = ptr->right;
			external[15]=1;
			t->pt[15] = bitbuild(ptr);
		}
	}
	//
	t->external = external;
	super_node++;
	return t;
}
////////////////////////////////////////////////////////////////////////////////////
void bitsearch(unsigned int ip,FILE *f){
	int i,j;
	unsigned int pre = 256;
	int prefix;
	int index;
	supertrie current = superoot;
	btrie ptr = root, temp=NULL;
	for(i=3;i<32;i+=4){
		index=0;
		prefix = (ip>>(31-i)) & 15;
		if(current->external[prefix] == 1){
			///pre computing
			if(current->internal[index]==1)
			pre = current->nexthop[index];
			for(j=3;j>0;j--){
				if(prefix&(1<<j)){
					index = 2*index + 2;
					if(current->internal[index]==1)
					pre = current->nexthop[index];
				}
				else {
					index = 2*index + 1;
					if(current->internal[index]==1)
					pre = current->nexthop[index];
				}
			}
			current = current -> pt[prefix];
		}//if
		else{
			if(current->internal[index]==1)
			pre = current->nexthop[index];
			for(j=3;j>0;j--){
				if(prefix&(1<<j)){
					index = 2*index + 2;
					if(current->internal[index]==1)
					pre = current->nexthop[index];
				}
				else {
					index = 2*index + 1;
					if(current->internal[index]==1)
					pre = current->nexthop[index];
				}
			}
		break;
		}//else
	}//for
	//fprintf(f,"%d\n",pre);
	//printf("%d\n",pre);
}
////////////////////////////////////////////////////////////////////////////////////
void set_table(char *file_name){
	FILE *fp;
	int len;
	char string[100];
	unsigned int ip,nexthop;
	fp=fopen(file_name,"r");
	while(fgets(string,50,fp)!=NULL){
		read_table(string,&ip,&len,&nexthop);
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
		read_table(string,&ip,&len,&nexthop);
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
	for(i=0;i<num_entry;i++)
		add_node(table[i].ip,table[i].len,table[i].port);
	end=rdtsc();
}
////////////////////////////////////////////////////////////////////////////////////
void count_node(btrie r){
	if(r==NULL)
		return;
	count_node(r->left);
	N++;
	count_node(r->right);
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
		printf("%d,%d\n", (i + 1) * 100, NumCntClock[i]);
	}
	return;
}
void IP_random(){
	int i, j;
	unsigned int temp;

	for(i = 0; i < num_query; i++){
		j = rand()%num_query;
		//printf("%d\n",j);
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
	int k ;
	FILE *f = fopen("test.txt","w+t");
	//set_query(argv[2]);
	//set_table(argv[1]);
	set_query("ip.txt");
	set_table("ip.txt");
	//set_query("IPv4_642161.txt");
	//set_table("IPv4_642161.txt");
	//set_query("IPv4_532089.txt");
	//set_table("IPv4_532089.txt");
	create();
	IP_random();
	begin = rdtsc();
	superoot = bitbuild(root);
	end= rdtsc();
//	printf("Size:%d\n",sizeof(linker));
	printf("Avg. Inseart: %llu\n",(end-begin)/num_entry);
	printf("number of nodes: %d\n",super_node);
	printf("Total memory requirement: %d KB\n",((super_node*sizeof(linker))/1024));
	////////////////////////////////////////////////////////////////////////////a
	IP_random();	
	for(j=0;j<100;j++){
		for(i=0;i<num_query;i++){
			begin=rdtsc();
			bitsearch(query[i],f);
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
	
	////////////////////////////////////////////////////////////////////////////
	//count_node(root);
	//printf("There are %d nodes in binary trie\n",N);
	return 0;
}
```

#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <string.h>





#define BLOCKSIZE 	1024
#define SIZE		1024*1000
#define END		0xffff
#define FREE		0
#define NOTFREE		1
#define ROOTBLOCKNUM	2
#define MAXOPENFILE	10

#define MOVE		0
#define FILL		1

#define ROW		40
#define COL		25

struct FCB
{
	unsigned long fd;		//文件描述符
	char filename[8];		//文件名
//	char exname[3];			//文件扩展名
	unsigned short attribute;	//0：目录文件 1：数据文件
	unsigned short hour;		//文件创建时间
	unsigned short min;		//文件创建时间
	unsigned short mon;		//文件创建日期
	unsigned short day;		//文件创建日期
	unsigned short first;		//文件起始盘块号
	unsigned long length;		//文件长度（字节）
	char free;			//表示目录项是否为空，0：空 1：非空
};

struct USEROPEN
{
/*	unsigned long fd;		//文件描述符
	char filename[8];		//文件名
	char exname[3];			//文件扩展名
	unsigned char attribute;	//0：目录文件 1：数据文件
	unsigned short time;		//文件创建时间
	unsigned short date;		//文件创建日期
	unsigned short first;		//文件起始盘块号
	unsigned long length;		//文件长度（字节）
*/

	struct FCB* file;	
	//文件使用中的动态信息
	char dir[80];			//打开文件所在的路径
	int count;			//读写指针
	char fcbstate;			//是否被修改 1：修改 0：未修改
	char topenfile;			//打开表项是否为空 0：空 1：非空
};

struct BLOCK0
{
	char info[200];			//存储一些描述信息
	unsigned short root;		//根目录文件的起始盘块号
	unsigned char* startblock;	//虚拟磁盘上数据区开始的位置
};

struct bitGraph
{
	short graph[ROW][COL];
};

struct FAT
{
	unsigned short id;		//0：FAT1 1：FAT2
	long table[SIZE];		//FAT表
};


unsigned char* myvhard;				//指向虚拟磁盘的起始地址
struct USEROPEN openfilelist[MAXOPENFILE];	//用户打开文件表数组
int curdir;					//当前目录的文件描述符fd
char currentdir[80];				//当前目录的目录名（包括目录的路径）
unsigned char* startp;				//记录虚拟磁盘上数据区的开始位置
unsigned char* FAT1;				//FAT1的起始位置
unsigned char* FAT2;				//FAT2的起始位置
unsigned char* block0;				//引导块起始位置
struct bitGraph* bitgraph;			//位示图
long fd = 0;					//文件描述符
struct FCB* root;				//根目录FCB
struct FCB* curfcb;				//当前目录的FCB

void init_bitgraph()
{
	bitgraph = (struct bitGraph*)malloc(sizeof(struct bitGraph));
	for(int i=0;i<7;i++)
		(bitgraph -> graph[0][i]) = NOTFREE;
	for(int i=7;i<COL;i++)
		(bitgraph -> graph[0][i]) = FREE;
	for(int i=1;i<ROW;i++)
		for(int j=0;j<COL;j++)
			(bitgraph -> graph[i][j]) = FREE;
}

/*
 * 更新位示图，参数为磁盘块号和操作码
 */
void update_bitgraph(int block, int flag)
{
	int i, j;
	if(flag > SIZE/BLOCKSIZE)
	{
		printf("block out of range !\n");
		return;
	}
	i = (block-1) / ROW;
	j = (block-1) % COL;
	if(flag == FILL)
		bitgraph -> graph[i][j] = NOTFREE;
	else if(flag == MOVE)
		bitgraph -> graph[i][j] = FREE;
	else
	{
		printf("failed when update bitgraph because of wrong flag\n");
		return;
	}
}

/*
 * 返回位示图中第一个空闲盘块号
 */
int get_free_block()
{
	for(int i = 0;i < ROW;i++)
		for(int j = 0;j < COL;j++)
			if(bitgraph -> graph[i][j] == FREE)
				return ROW*i + j + 1;
}

void init_FAT()
{
	//一开始把FAT表的所有表项初始化为END
	for(int i=0;i < SIZE/BLOCKSIZE;i++)
	{
		((struct FAT*)FAT1) -> table[i] = END;
		((struct FAT*)FAT2) -> table[i] = END;
	}

	((struct FAT*)FAT1) -> table[0] = END;
	((struct FAT*)FAT1) -> table[1] = 2;
	((struct FAT*)FAT1) -> table[2] = END;
	((struct FAT*)FAT1) -> table[3] = 4;
	((struct FAT*)FAT1) -> table[4] = END;
	((struct FAT*)FAT1) -> table[5] = 6;
	((struct FAT*)FAT1) -> table[6] = END;
	
	((struct FAT*)FAT2) -> table[0] = END;
	((struct FAT*)FAT2) -> table[1] = 2;
	((struct FAT*)FAT2) -> table[2] = END;
	((struct FAT*)FAT2) -> table[3] = 4;
	((struct FAT*)FAT2) -> table[4] = END;
	((struct FAT*)FAT2) -> table[5] = 6;
	((struct FAT*)FAT2) -> table[6] = END;
}


/*
 * 当盘块不够时新分配一个
 * 当盘块太多时回收一个
 */
void update_FAT(struct FCB* file, int flag)
{
	int i = file -> first;
	int j = i;
	while(((struct FAT*)FAT1) -> table[i] != END)
	{
		j = i;
		i = ((struct FAT*)FAT1) -> table[i];
	}
	if(flag == FILL)
	{
		int free = get_free_block();
		((struct FAT*)FAT1) -> table[i] = free;
		((struct FAT*)FAT1) -> table[free] = END;
		((struct FAT*)FAT2) -> table[i] = free;
		((struct FAT*)FAT2) -> table[free] = END;
	}
	else if(flag == MOVE)
	{
		update_bitgraph(i, MOVE);
		((struct FAT*)FAT1) -> table[j] = END;
		((struct FAT*)FAT2) -> table[j] = END;
	}
}

/*
 * 传入参数是盘块号
 * 返回该盘块号的起始地址
 */
char* get_add(int block)
{
	return myvhard + block * BLOCKSIZE;
}

struct FCB* get_current_FCB()
{
	int i;
	for(i = 0;(openfilelist[i].file)->fd != curdir;i++);
	return openfilelist[i].file;
}

void startsys()
{
	
}

/*
 * 格式化虚拟磁盘，给引导块、FAT1、FAT2和根目录分配磁盘块
 * 同时初始化FAT1、FAT2和位示图
 */
void my_format()
{
	time_t ptime;
	struct tm* p;
	time(&ptime);
	p = gmtime(&ptime);

	//分配虚拟磁盘空间
	myvhard = (char*)malloc(SIZE);
	block0 = myvhard;					//initial
	FAT1 = block0 + BLOCKSIZE;
	FAT2 = FAT1 + 2 * BLOCKSIZE;
	startp = FAT2 + 2 * BLOCKSIZE;

	((struct FAT*)FAT1) -> id = 0;				//init FAT
	((struct FAT*)FAT2) -> id = 1;	
	init_FAT();
	
	//初始化位示图
	init_bitgraph();

	//初始化引导块
	((struct BLOCK0*)block0) -> root = 5;			//盘块从0开始编号
	((struct BLOCK0*)block0) -> startblock = startp;

	//初始化根目录FCB
	root = (struct FCB*)malloc(sizeof(struct FCB));
	root -> fd = fd++;
	strcpy(root -> filename, "root");
	strcpy(root -> filename, "");
	(root -> attribute) = 0;
	(root -> hour) = p -> tm_hour;
	(root -> min) = p -> tm_min;
	(root -> mon) = p -> tm_mon;
	(root -> day) = p -> tm_mday;
	(root -> first) = 5;
	(root -> length) = 2*sizeof(struct FCB);
	(root -> free) = 0;
	
	char* first_add = get_add(5);
	struct FCB* t = (struct FCB*)first_add;
	strcpy(t->filename,".");
	t -> attribute = 0;
	(t)->hour = p->tm_hour;
	(t)->min = p->tm_min;
	(t)->mon = p->tm_mon;
	(t)->day = p->tm_mday;
	(t)->first = 5;
	(t)->length = sizeof(struct FCB);
	(t)->free = 0;

	strcpy((t+1)->filename,"..");
	(t+1) -> attribute = 0;
	(t+1)->hour = p->tm_hour;
	(t+1)->min = p->tm_min;
	(t+1)->mon = p->tm_mon;
	(t+1)->day = p->tm_mday;
	(t+1)->first = 5;
	(t+1)->length = root->length;
	(t+1)->free = 0;
	

	//初始化openfilelist
	openfilelist[0].file = root;
	strcpy(openfilelist[0].dir, "/");
	openfilelist[0].count = 0;
	openfilelist[0].fcbstate = 0;
	openfilelist[0].topenfile = 0;
	
	strcpy(currentdir, " /");
	curdir = 0;
	curfcb = root;
}

void my_cd(char* dirname)
{
	char* first_add = get_add(curfcb -> first);
	int i;
	struct FCB* t = (struct FCB*)first_add;
	for(i = 0;i < (curfcb->length)/sizeof(struct FCB);i++)
	{
		if(strcmp(dirname, (t+i)->filename) == 0)
		{
			if((t+i)->attribute != 0)
			{
				printf("%s is not a directory !\n",dirname);
				return;
			}
			if(strcmp((t+i)->filename,"..") == 0)
			{
				int l = strlen(currentdir);
				//*(currentdir+l2-l1-1) = '\0';
				for(int j = l-2;j > 0;j--)
				{
					if(*(currentdir+j) == '/')
					{
						*(currentdir+j+1) = '\0';
						break;
					}
				}
				curfcb = (t+i);
				curdir = curfcb -> fd;
			}
			else if(strcmp((t+i)->filename,".") == 0)
			{
				curdir = curfcb -> fd;
			}
			else
			{
				curfcb = (t+i);
				curdir = curfcb -> fd;
				strcat(currentdir, dirname);
				strcat(currentdir, "/");
			}
			printf("present working directory: %s\n",currentdir);
			return;
		}
	}
	printf("there is no directory called %s !\n",dirname);
}

void my_mkdir(char* dirname)
{
	char* first_add = get_add(curfcb -> first);
	time_t ptime;
	struct tm* p;
	time(&ptime);
	p = gmtime(&ptime);
	int i;
	int block;

	//检查是否有重名文件
	struct FCB* t = (struct FCB*)first_add;		//t是当前目录所对应盘块号的首地址
	for(i = 0;i < (curfcb->length/sizeof(struct FCB));i++)
	{
		if(strcmp(dirname, (t+i)->filename) == 0)
		{
			printf("the directory is already exist\n");
			return;
		}
	}

	//分配盘块号	
	block = get_free_block();


	//在FCB中填入基本信息
	strcpy((t+i)->filename, dirname);
	(t+i)->attribute = 0;
	(t+i)->hour = p->tm_hour;
	(t+i)->min = p->tm_min;
	(t+i)->mon = p->tm_mon;
	(t+i)->day = p->tm_mday;
	(t+i)->first = block;
	(t+i)->length = 2*sizeof(struct FCB);		//初始有两个目录项，.和..
	(t+i)->free = 0;

	//更新位示图
	update_bitgraph(block,FILL);

	//更新当前目录的信息	
	curfcb->length += sizeof(struct FCB);
	curfcb->free = 1;
	
	//将该目录下所有子目录的..长度信息更新
	for(i = 2;i < (curfcb->length/sizeof(struct FCB));i++)
	{
		if((t+i) -> attribute == 0)
		{
			(((struct FCB*)get_add((t+i)->first))+1) -> length = curfcb -> length;
		}
	}

	//在新的目录块中填入.和..的信息
	first_add = get_add(block);
	t = (struct FCB*)first_add;
	strcpy(t->filename,".");
	t -> attribute = 0;
	(t)->hour = p->tm_hour;
	(t)->min = p->tm_min;
	(t)->mon = p->tm_mon;
	(t)->day = p->tm_mday;
	(t)->first = block;
	(t)->length = 2*sizeof(struct FCB);
	(t)->free = 0;

	strcpy((t+1)->filename,"..");
	(t+1) -> attribute = 0;
	(t+1)->hour = p->tm_hour;
	(t+1)->min = p->tm_min;
	(t+1)->mon = p->tm_mon;
	(t+1)->day = p->tm_mday;
	(t+1)->first = curfcb->first;
	(t+1)->length = curfcb->length;
	(t+1)->free = 1;

}

void my_ls()
{
	char* first_add = get_add(curfcb->first);
	struct FCB* t = (struct FCB*)first_add;
	for(int i = 0;i < (curfcb->length)/sizeof(struct FCB);i++)
	{
		if(strcmp((t+i) -> filename, "..") == 0 || strcmp((t+i) -> filename, ".") == 0)
			printf("%s\n",(t+i)->filename);
		else
			printf("%s 			%d	%d月 %d日 %d:%d\n",(t+i)->filename,(t+i)->attribute,(t+i)->mon,(t+i)->day,(t+i)->hour,(t+i)->min);
	}
}

void my_create(char* fname)
{
	char* first_add = get_add(curfcb->first);
	struct FCB* t = (struct FCB*)first_add;
	int i;

	time_t ptime;
	struct tm* p;
	time(&ptime);
	p = gmtime(&ptime);

	for(i = 0;i < curfcb->length/sizeof(struct FCB);i++)
	{
		if(strcmp((t+i)->filename, fname) == 0)
		{
			printf("the file is already exists\n");
			return;
		}
	}

	//更新当前目录的信息，新增加一个FCB
	strcpy((t+i)->filename, fname);
	(t+i) -> attribute = 1;
	(t+i) -> mon = p -> tm_mon;
	(t+i) -> day = p -> tm_mday;
	(t+i) -> hour = p -> tm_hour;
	(t+i) -> min = p -> tm_min;
	(t+i) -> length = 0;
	(t+i) -> free = 0;

	//分配一个空闲块
	int block = get_free_block();

	(t+i) -> first = block;
	t = (struct FCB*)get_add(block);

	//更新当前目录
	curfcb -> length += sizeof(struct FCB);

	//更新位示图和FAT表
	update_bitgraph(block, FILL);
	update_FAT(t, FILL);

	t = (struct FCB*)first_add;	
	for(i = 2;i < (curfcb->length/sizeof(struct FCB));i++)
	{
		if((t+i)->attribute == 0)
		{
			(((struct FCB*)get_add((t+i)->first))+1) -> length = curfcb -> length;
		}
	}
}

void my_rm(char* fname)
{
	char* first_add = get_add(curfcb -> first);
	struct FCB* t = (struct FCB*)first_add;
	int i;
	for(i = 0;i < curfcb->length/sizeof(struct FCB);i++)
	{
		if(strcmp((t+i)->filename, fname) == 0)
		{
			if((t+i) -> attribute == 0)
			{
				printf("%s is a directory\n",fname);
				return;
			}
			
			update_bitgraph((t+i)->first, MOVE);
			update_FAT((t+i), MOVE);
			int length = (curfcb->length)/sizeof(struct FCB);
			//把磁盘块中最后一个文件的FCB复制到删除的文件的位置上
			strcpy((t+i)->filename,(t-1+length)->filename);
			(t+i) -> attribute = (t-1+length) -> attribute;
			(t+i) -> mon = (t-1+length) -> mon;
			(t+i) -> day = (t-1+length) -> day;
			(t+i) -> hour = (t-1+length) -> hour;
			(t+i) -> min = (t-1+length) -> min;
			(t+i) -> first = (t-1+length) -> first;
			(t+i) -> length = (t-1+length) -> length;
			(t+i) -> free = (t-1+length) -> free;


			(curfcb -> length) -= sizeof(struct FCB);
			return;
		}
	}
	printf("there is no file called %s\n",fname);
}

void my_rmdir(char* dirname, struct FCB* curfcb)
{
	char* first_add = get_add(curfcb -> first);
	struct FCB* t = (struct FCB*)first_add;
	int i;
	for(i = 0;i < curfcb->length/sizeof(struct FCB);i++)
	{
		if(strcmp((t+i)->filename, dirname) == 0)
		{
			if((t+i)->length/sizeof(struct FCB) <= 2)
			{
				update_bitgraph((t+i)->first, MOVE);
				update_FAT((t+i),MOVE);
				int length = (curfcb->length)/sizeof(struct FCB);
				//把磁盘块中最后一个文件的FCB复制到删除的文件的位置上
				strcpy((t+i)->filename,(t-1+length)->filename);
				(t+i) -> attribute = (t-1+length) -> attribute;
				(t+i) -> mon = (t-1+length) -> mon;
				(t+i) -> day = (t-1+length) -> day;
				(t+i) -> hour = (t-1+length) -> hour;
				(t+i) -> min = (t-1+length) -> min;
				(t+i) -> first = (t-1+length) -> first;
				(t+i) -> length = (t-1+length) -> length;
				(t+i) -> free = (t-1+length) -> free;
				
				(curfcb -> length) -= sizeof(struct FCB);
				return;
			}
			else
			{
				struct FCB* new_t = (struct FCB*)get_add((t+i)->first);
				for(int j = 2;j < (new_t+i)->length/sizeof(struct FCB);i++)
				{
					if((new_t+j)->attribute == 1)
					{
						update_bitgraph((new_t+j)->first, MOVE);
						update_FAT((new_t+j), MOVE);
						(t+i) -> length -= sizeof(struct FCB);
					}
					else
						my_rmdir((new_t+j)->filename, (t+i));
				}
				update_bitgraph((t+i)->first, MOVE);
				update_FAT((t+i),MOVE);
				int length = (curfcb->length)/sizeof(struct FCB);
				//把磁盘块中最后一个文件的FCB复制到删除的文件的位置上
				strcpy((t+i)->filename,(t-1+length)->filename);
				(t+i) -> attribute = (t-1+length) -> attribute;
				(t+i) -> mon = (t-1+length) -> mon;
				(t+i) -> day = (t-1+length) -> day;
				(t+i) -> hour = (t-1+length) -> hour;
				(t+i) -> min = (t-1+length) -> min;
				(t+i) -> first = (t-1+length) -> first;
				(t+i) -> length = (t-1+length) -> length;
				(t+i) -> free = (t-1+length) -> free;

				(curfcb -> length) -= sizeof(struct FCB);
				return;
			}
		}
	}
	printf("no such directory\n");
}

void my_shell()
{
	char command[20];
	char t[80];
	while(1)
	{
		scanf("%s",command);
		if(strcmp(command, "my_mkdir") == 0)
		{
			printf("input the directry name:\n");
			scanf("%s",t);
			my_mkdir(t);
		}
		else if(strcmp(command, "my_ls") == 0)
			my_ls();
		else if(strcmp(command, "my_cd") == 0)
		{
			printf("input the directry name:\n");
			scanf("%s",t);
			my_cd(t);
		}
		else if(strcmp(command, "my_create") == 0)
		{
			printf("input filename:\n");
			scanf("%s",t);
			my_create(t);
		}
		else if(strcmp(command, "my_rm") == 0)
		{
			printf("input filename:\n");
			scanf("%s",t);
			my_rm(t);
		}
		else if(strcmp(command, "my_rmdir") == 0)
		{
			printf("input filename:\n");
			scanf("%s",t);
			my_rmdir(t,curfcb);
		}
		else if(strcmp(command, "my_exit") == 0)
		{
			return;
		}
		else
			printf("command not found\n");
	}
}


int main()
{
	my_format();
	my_shell();
	return 0;
}
int read(int fid){
	char *text = (char *)malloc(2*BLOCKSIZE);
	FCB *file;
	FAT *FAT1;
	int i = 0;
	int len = openfilelist[fileopenptr].length;

	if(fid > MAXOPENFILE){
		if(fileopenptr > MAXOPENFILE){
			printf("file resrouce id is error , may be you have not open it");
		}
		fid = fileopenptr;
	}
	if(openfilelist[fileopenptr].attribute != 0x5){
		printf("have no opened file or the file you opened can't write!\n");
		return -1;
	}

	if(openfilelist[fileopenptr].length < BLOCKSIZE){
		len = openfilelist[fileopenptr].length;
		text = (char *)(vhard + openfilelist[fileopenptr].first * BLOCKSIZE);
		i = 0;
		printf("Read Mode | file length:%d\n", len);
		while((len --) > 0){
			putchar(text[i ++]);
		}
		printf("\n");
	}else {
		int id = 0;
		len = openfilelist[fileopenptr].length;
		FAT1 = (FAT*)(vhard + BLOCKSIZE + openfilelist[fileopenptr].first * sizeof(FAT));
		text = (char *)(vhard + openfilelist[fileopenptr].first * BLOCKSIZE);
		for(i = 0;i <= BLOCKSIZE; i++){
			putchar(text[i]);
		}
		while(FAT1->id != END){
			int limit;
			if(FAT1->id == FREE){
				printf("FAT error!\n");
				return -1;
			}
			len -= BLOCKSIZE;
			limit = len < BLOCKSIZE?len:BLOCKSIZE;
			text = (char *)(vhard + FAT1->id * BLOCKSIZE);
			for(i = 0;i<= limit; i++){
				putchar(text[i]);
			}
			FAT1 = (FAT *)(vhard + BLOCKSIZE + FAT1->id * sizeof(FAT));
		}
	}
	return 0;
}













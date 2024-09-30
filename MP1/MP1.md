

#SC_Halt

Halt() 是一個 API，宣告在 syscall.h 中(只有宣告沒有實作)，編譯時，nachOS 會去找Halt的實作方式，也就是system call，
就是test/start.S，裡面是用組合語言寫的。然後將他們連結起來，最後轉成binary code。這應該是make halt在做的事。
所以編譯完後，原本Halt()的位置，會變成一個system call的符號，連type也會存好(是halt)。這時../build_linux/nachos -e halt，回敬一連串初始化,
如new kernel...，最後執行到 Machine::Run()

Machine::Run()主要是模擬mips的硬體，一條條執行機器碼。其中for loop 內的 OneInstruction(instr); 會一條一條執行instruction，
注意，instr只是一個容器，在OneInstruction(instr)中，才會去memory讀指令，並把decode 後的指令存在instr中 OneInstruction(instr)

OneInstruction(instr)，裡面依照program counter，去memory將指令讀出來，經過decode，得到op code, rs, rt, rd, imm等資訊，用switch case 來
比對是什麼指令，以halt的例子來說(記得編譯時，已經把Halt()這行換成system call 的符號)，對到的case 會是case OP_SYSCALL，
裡面執行的程式為RaiseException(SyscallException, 0)，也就是說現在要raise exception(有很3種，systam call(user 主動call), 
exception(segmentation fault, divideed by 0, ...), interrupt(硬體錯誤))，然後傳入說是system call 的exception。

RaiseException(SyscallException, 0)裡會呼叫 kernel->interrupt->setStatus(SystemMode)，也就是先設成system mode，
再執行 ExceptionHandler(which); (which就是 RaiseException(SyscallException, 0)中的SyscallException而已，再傳入給下一個函式判斷)，

ExceptionHandler(which)中，就會判斷這個which是什麼，有可能有三種(system call, exception, interrupt)，但可發現nachos只有實作system call，
所以其他就會報錯。現在確定是system call了，接著就確認是什麼system call，它會執行int type = kernel->machine->ReadRegister(2);意思是，
reg2 讀是什麼樣的system call，記得編譯時，會將reg2的值設為0(代表SC_Halt)，所以接著的switch case 就會跳到SC_Halt，裡面執行SysHalt()

SysHalt()裡會執行，kernel->interrupt->Halt()，(kernel->interrupt 存的是一個interrupt類別);

再去interrupt底下看interrupt->Halt()是什麼，結果是delete kernel，就是將kernel 釋放，因此整個nachos的程式就結束了。


#SC_Create

一樣編譯時，start.S回將Create()換成System call的符號，並將system call的type存在reg2中，
前面run()->OneInstruction(instr)->RaiseException(SyscallException, 0)->ExceptionHandler(which)都和Halt()一樣，

但在ExceptionHandler(which)中，判斷是什麼System call 時，會跳到SC_Create，
裡面會執行val = kernel->machine->ReadRegister(4)，
和 char *filename = &(kernel->machine->mainMemory[val]);就是將要開啟檔案的檔名讀出，
接著執行 status = SysCreate(filename)，

SysCreate(filename)，會呼叫kernel->fileSystem->Create(filename)，去filesys.h查看，發現有兩個Create，
一個是#ifdef FILESYS_STUB下的Create，也就是預設的，現在MP1作業使用的，直接call unix 實作。另一個是#else下的Create，也就是要自己實作了，
可能是之後的作業內容。

再仔細看 #ifdef FILESYS_STUB下的Create，裡面呼叫sysdep.cc 底下 的OpenForWrite(name)，照sysdep.cc的說明，這個header是unix system call 的
interface ，所以之後就交給unix 處理了。

再回到ExceptionHandler(which)中，發現執行完 status = SysCreate(filename)後，還有一些指令，像是，    
kernel->machine->WriteRegister(2, (int)status);應該就是把新建檔案成功或失敗(1 or 0)寫入reg2中，接著執行了以下3行指令，

kernel->machine->WriteRegister(PrevPCReg, kernel->machine->ReadRegister(PCReg));
kernel->machine->WriteRegister(PCReg, kernel->machine->ReadRegister(PCReg) + 4);
kernel->machine->WriteRegister(NextPCReg, kernel->machine->ReadRegister(PCReg) + 4);

簡而言之就是將 PC + 4，SC_Halt沒用到，我想是因為，接著就要結束程式，也不必再存PC了。


#SC_PrintInt()

case SC_PrintInt中，執行 val = kernel->machine->ReadRegister(4)，先去reg4讀取參數( 就是PrintInt(result)中的result )，
接著呼叫 sysPrintInt(val);

sysPrintInt(val)中會執行 kernel->synchConsoleOut->PutInt(val);

kernel->synchConsoleOut->PutInt(val)中會執行 consoleOutput->PutChar(str[idx])

consoleOutput->PutChar(str[idx]) 這行其實就是模擬印到螢幕上，裡面會呼叫WriteFile(writeFileNo, &ch, sizeof(char))，
其中WriteFile()定義在sysdep.cc，是nachos要call unix 的interface 的interface，(就是WriteFile中會call write，而write 才是定義在ostream中，unix的interface)。總而言之，WriteFile(writeFileNo, &ch, sizeof(char))可以當成是印一個字元到螢幕上。接下去，會呼叫kernel->interrupt->Schedule(this, ConsoleTime, ConsoleWriteInt)，其中ConsoleTime是100，也就是它會模擬剛剛印一個字元會花100個tick，然後現在會預定一個interrupt，在100個tick後發生(實際上，應該是硬體發現io做完了，再產生interrupt；nachos是模擬，所以它是站在一個全知的角度，預定未來發生的事)

然後，重要的是，我覺得kernel->interrupt->Schedule 和 OS中的排程(scheduler)沒關係!!! 以下在Interrupt::Schedule中寫的註解，
//	NOTE: the Nachos kernel should not call this routine directly.
//	Instead, it is only called by the hardware device simulators.
它說這個interrupt->Schedule只能由hardware呼叫，不能由Nachos kernel呼叫，也就是說，這個interrupt->shedule是在模擬hardware interrupt的發生，
和short-term scheduler沒有關係!

看interrupt->schedule 第一行，int when = kernel->stats->totalTicks + fromNow;
可發現 when 就是現在的total ticks 加 剛剛的ConsoleTime 100，
PendingInterrupt *toOccur = new PendingInterrupt(toCall, when, type);
它會new 一個PendingInterrupt的物件，之後加到pending 這個list中

值得注意的是，
new PendingInterrupt(toCall, when, type)中 toCall 是 consoleOutput(這個說是模擬serial port，我也不太清楚，總之先當成輸出字元到螢幕的連接線路好了，之後interrupt產生應該還要回到這個東西，印下一個字元)，when 是之後要interrupt 的時間，type 是指甚麼類型的interrupt，這裡是ConsoleWriteInt，即印一個int。

接下來是，
waitFor->P();
waitFor是一個Semephore的物件，P代表wait，也就是 s-=1 (假設一開始s = 1，同時只有一個人可以做事)，V代表signal，也就是s += 1，
所以waitFor->P()代表如果現在有人在使用(s = 0)，它就得等，直到有人waitFor->V()，此時s += 1，變成1，才可以執行下一個consoleOutput->PutChar(str[idx]);

之後就不太清楚發生什麼事，猜測應該是，目前要wait，所以kernel 就是做context switch，讓其他thread 執行，其他thread 就會執行......

依照提示，去看Machine::Run()，再看Machine::OneTick()，再看Interrupt::CheckIfDue()。我覺得Machine::Run()和Machine::OneTick()應該是不會執行，因為 由debug提供的message 可發現應該不是那樣(../build.linux/nachos -e add -d c)，但會執行到 Interrupt::CheckIfDue()，只是可能是從別的地方過來的，提示上也寫到
Note: The actual execution flow in NachOS may differ from the flow described
above, as NachOS is a simulated operating system. Explanation to the pseudo
function call above is sufficient enough.
但就不知道是 實際的機器是照提示跑嗎?還是提示只是要run->onetick->checkIfDue，以這個看起來合理的流程，也是MP1我們可以理解的方式，介紹CheckIfDue()。

總之，接下來要執行CheckIfDue()，
它會將pending list 裡的interrupt 拿出來執行，當然前面會檢查一下，時間是不是到了，
接著比較重要的是 next->callOnInterrupt->CallBack();這是 ConsoleOutput 中的callback()，
接著再呼叫callWhenAvail->CallBack()，這是 SynchConsoleOutput 中的callback()，
然後就會執行 waitFor->V()，也就是額接回去，執行下一個consoleOutput->PutChar(str[idx])，
終於trace 完!!!



#Makefile
以執行halt為例，我們會在test目錄下輸入
make clean
make halt
../build.linux/nachos halt


其中 make clean 會去 Makefile中 找
clean:
	$(RM) -f *.o *.ii
	$(RM) -f *.coff
其中$(RM)是make 預設的變數，代表rm -f 即強制刪除，依照上面2行，會刪掉 .o, .ii, .coff 檔

make halt 會去找

halt: halt.o start.o
	$(LD) $(LDFLAGS) start.o halt.o -o halt.coff
	$(COFF2NOFF) halt.coff halt

可發現halt需要halt.o 和 start.o，所以make又會去找

halt.o: halt.c
	$(CC) $(CFLAGS) -c halt.c

start.o: start.S ../userprog/syscall.h
	$(CC) $(CFLAGS) $(ASFLAGS) -c start.S

halt.o 需要 halt.c，這時，halt.c 已經存在test目錄底下，因此可以執行下面的指令
$(CC) $(CFLAGS) -c halt.c

$(CC)是指gcc 
至於$(CFLAGS)，可從makefile上找到
CFLAGS = -g -G 0 -c $(INCDIR) -B/usr/bin/local/nachos/lib/gcc-lib/decstation-ultrix/2.95.2/ -B/usr/bin/local/nachos/decstation-ultrix/bin/
推測是gccc 後面接的參數，其他就不太清楚，可能是編成某種型態，讓之後的nachos可以執行
總之，halt.o 就編譯好了

start.o 需要start.S 和 ../userprog/syscall.h，可知目錄下都有這些檔案，
所以可以接著執行
$(CC) $(CFLAGS) $(ASFLAGS) -c start.S

其中start.S使用組合語言寫的
這邊值得注意的是，看起來像是gcc 也可以來編譯組合語言
總之也編譯好得到start.o

因此最一開始的
halt: halt.o start.o
	$(LD) $(LDFLAGS) start.o halt.o -o halt.coff
	$(COFF2NOFF) halt.coff halt
就可以接著執行，經過中間.coff(不清楚是什麼)
最後就得到 halt 這個執行檔，然後可以被nachos執行
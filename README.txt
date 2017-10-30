
подход event_loop
4 библиотеки для программирования сетевых приложений

1) libevent - наиболее старая, наиболее большая
2) libev - поновее
3) libuv - совсем новая

4) boost::asio

select
poll
epoll
kqueue

1)

libevent умеет:
devport
evport
win32

Создается объект, называемый event_base
в нем начинаем рег. различные события
событие event, к которму привязана функция event_callback,
когда произойдет, event_base вызовет эту функцию
так заполняется и создается в памяти event_base и запускается цикл (loop)
с этого момента мы теряем управление, и когда происходит какое-либо событие,
loop вызывает нужный нам callback и callback исполняется
callback исполнился - продолжился наш цикл

struct event_base *event_base_new (void);
// создает event_base

struct event_config *event_config_new(void);
// создает event_base с config
struct event_base *event_base_new_with_config
					(const struct event_config *cfg);

void event_config_free(struct event_config *cfg);

void event_base_free(struct event_base *base); // уничтожить base

запустим event_loop:

int event_base_loop(struct event_base *base, int flags);
для запуска event_loop мы должны зарегать хотя бы одно событие,
которое может случиться
если мы сняли с регистрации все события, event_loop завершается
однако мы можем менять поведение с помощью флагов

EVLOOP_ONCE - как только исполнилось 1 событие, цикл завершается
EVLOOP_NONBLOCK - мы никогда не находимся в цикле без дела
EVLOOP_NO_EXIT_ON_EMPTY - как только с регистрации снимается последнее
событие, он все равно продолжает выполняться (режим работает корректно, только
в многопоточных программах, когда мы можем остановить цикл из другого потока
либо зарегать новые события)

int event_base_dispatch (*base);
// то же что event_base_loop, только flags = 0

запустили цикл, как остановить:

1) вызвать int event_base_loopexit (*base, const struct timeval *tv);
 // она сработает после tv - таймаута, (NULL - немедленное выполнение)
 есть неск-ко callback которые должны быть исполнены -
 вызвав event_base_loop_exit мы выполним все callback и только потом выйдем
 из loop

2) int event_base_loopbreak(*base);
- функция дожидается исполнения текущего callback 
и даже если у нас их в очереди есть еще несколько, она завершает loop

как проверить, не запущены ли у нас в другом потоке или callback-e функции loopexit или loopbreak?

int event_base_got_exit(*base); // если запущен exit вернет !0
int event_base_got_break(*base); // аналогично с break

создание event;

struct event *event_new(struct event_base *base,
// event уже при создании привязан к event_base
// для каждого event_base мы создаем event-ы, которые регистрируем/снимаем с
// регистрации. Менять base event не может
	evutil_socket_t fd, // дескриптор сокета
	short what, // какое именно событие рег. (флаг)
	event_callback_fn cb,
	void *arg);

short what -
EV_TIMEOUT - выполнится после таймаута
EV_READ - ожидаем возможность читать и сокета
EV_WRITE
EV_SIGNAL - предн. для работы с сигналами
--------
EV_PERSIST - событие после выполнения останется навечно в base, пока мы
не удалим его явно
EV_ET - edge-triggered-режим

event_callback_fn -
typedef void (*event_callback_fn) (evutil_socket_t,
				short,
				void*);

void *arg -
	void *event_self_cbarg();
	например мы создали EV_PERSIST event, но в какой-то момент нам нужно
	снять его с регистрации в callback-е
	мы его передаем в cbarg и в callbacke мы имеем указатель на сам
	event который можем снять с регистрации

как устанавливать и снимать event с регистрации

	int event_add (struct event *ev, const struct timeval *tv); // вкл
    int event_del (struct event *ev); // выкл

осв. памяти под event
void event_free(struct event *event);

помимо непоср. работы с сетью, libevent содержит вспомогательные ср-ва

буфер

evbuffer - похож на std::deque

struct evbuffer *evbuffer_new (void);
void evbuffer_free(*buf);

состоит из кусочков данных в виде очереди

evbuffer_add
_add_printf
_add_vprintf
_add_reference
_add_buffer
_add_buffer_reference
_add_file/file_segnent
_read/readln

evbuffer_prepend
_prepend_buffer

evbuffer_get_length
evbuffer_get_contiguous_space

evbuffer_pullup - несколько блоков объединяет в один

evbuffer_drain - удалить не читая
_remove
_remove_buffer

evbuffer_copyout
_copyout_from
_write
_write_atmost

evbuffer_search
_search_range
_search_eol

evbuffer_peek
evbuffer_reserve_space
_commit_space



bufferevents - содержит 2 evbuffer

socket_based bufferevents - пара evbuffer и набор событий
Обеспечивает буферизированный ввод-вывод для сокета
(мы отправляем пачку данных в сокет, а сокет недоступен для
записи, и мы храним это дело в буфере, как только сокет становится
доступным, мы отправляем нашу пачку)

struct bufferevent* bufferevent_socket_new(struct event_base* base,
	evutil_socket_t fd,
	struct bufferevent_options options);
void buffer_event_free(struct bufferevent* bev);

int bufferevent_socket_connect(struct bufferevent* bev,
							struct sockaddr* address,
							int addrlen); // соединение

события, котторые в eventloop можно отследить:

BEV_EVENT_READING
_WRITING
_ERROR
_TIMEOUT
_EOF
_CONNECTED


создали, соединились, что дальше?

можем по buffereent получить evbuffer:
struct evbuffer* bufferevent_get_input
								_output
								(struct bufferevent* bufev);

int bufferevent_write(*bufev, const void* data, size_t size);
int bufferevent_read(*bufev, void* data, size_t size);

связать с др. bufferevenr

int bufferevent_write_buffer(*buffer - наш,
					struct evbuffer* buf - не наш);

int bufferevent_read_buffer(*buffer, struct evbuffer* buf);

вытолкнуть в сокет:
int bufferevent_flush(*bufev, short iotype,
					enum bufferevent_flush_state flush);
bufferevent_set_timeouts(*bufev, const struct timeval* timeout_read,
						const struct timeval* timout_write);


серверные сокеты:

struct evconnlistener* evconnlistener_new(
		struct event_base* base,
		evconnlistener_cb cb,
		void* ptr, // параметр от callback'a
		unsigned flags, int backlog,
		evutil_socket_t fd);

socket->bind->listen->loop(..){accept}

fd создан в socket
-> fd может находиться после bind или listen
backlog - 2-й параметр listen (SOMAXCONN например)

struct evconnlistener* evconnlistener_new_bind(
				struct event_base* base,
				evconnlistener_cb cb,
				void* ptr,
				unsigned int flags, int backlog,
				struct sockaddr* sa - мастер, int socklen);
- создается сам сокет

typedef void (*evlistener_cb)(struct evlistener* listener,
					evutil_socket_t sock,
					struct sockaddr *addr, // сокет клиента
					void* ptr);

void evconnlistener_free(*listener);




libev - более молодой проект, быстрее, понятнее, но нет http-серверов

event_base = ev_loop
event = ev_io

остальное точно так же

struct ev_loop* ev_default_loop(unsigned int flags);
			ev_default_destroy();

struct ev_loop* ev_loop_new(unsigned int flags);
			ev_loop_destroy(loop);

int ev_is_default_loop(loop);

unsigned int ev_loop_count(loop); // счетчик событий в loop

ev_loop(loop, flags); // запуск loop
ev_unloop(loop, how);// остановка loop

если флаги 0, то loop бесконечен (пока не unloop)

флаги:
EVLOOP_NONBLOCK
EVLOOP_ONESHOT

struct ev_io - watcher

анатомия watcher'a
(взято из док-ии libev)

struct ev_loop* loop = ev_default_loop(0);
/*	stdin = 0
	stdout = 1
	stderr = 2	*/
вместо дескриптора дадим ev_io (watcher'у) stdin (0)
- будем слушать ввод с клавиатуры

struct ev_io stdin_watcher;
ev_init(stdin_watcher, my_cb);
ev_io_set(&stdin_watcher, STDIN_FILENO /* = 0 */, EV_READ);
далее watcher нужно добавить в loop
ev_io start(loop, &stdin_watcher);
ev_loop(loop, 0); // цикл запущен
осталось написать callback (my_cb)

static void my_cb(*loop, *w, int revents){
мы можем иметь один callback для разных типов событий
в отличие от lebevent.

внутри callback мы останавливаем watcher
ev_io_stop(watcher);
выключаем loop:
ev_unloop(loop, EVUNLOOP_ALL);
}

у callback'а при вызове не указывается никаких аргументов
	-> ev_init(stdin_watcher, my_cb /*(пустота)*/);

как эти аргументы передать?
сделаем свой watcher
у нас есть тип ev_io

struct my_io	{
	struct ev_io io;
	int value1;
	void* value2;
	............
}

адреса структуры my_io и адрес переменной ev_io io -
совпадают (выравнивание)

наша структура будет 1-ым арг-том в ev_init - (stdin_watcher)
затем она перейдет в ev_io_set(&stdin_watcher...)
а затем в callback: my_cb(..., *w, ..)
тогда мы можем внутри callback'a привести w к типу my_io:

struct my_io *w_ = (struct my_io*)w;t, 60

таким образом мы связали некоторую память с watcher'ом

сейчас познакомимся с watcher'ами, реализующими поведение
таймера:

struct ev_timer t;
ev_timer_init (&t, my_cb, /*a = */ 60.,/*b=*/ 0.);
a - когда сработает таймер
b - когда повторится
 если 0, то не повторится
 если !0, то будет срабатывать через каждые b секунд

ev_timer_start(loop, &t);

// таймер запустился, вызовется callback

static void my_cb(*loop, *w /*наш таймер */, revents)	{
	//..
}



ev_periodic - тоже таймеры, но базируются на абсолютном
времени

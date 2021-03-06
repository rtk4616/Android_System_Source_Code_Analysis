好几个月都没有更新过博客了，从今天开始，老罗将尝试对[Android](http://lib.csdn.net/base/android)系统的UI实现作一个系统的分析，也算是落实之前所作出的承诺。提到[android](http://lib.csdn.net/base/android)系统的UI，我们最先接触到的便是系统在启动过程中所出现的画面了。Android系统在启动的过程中，最多可以出现三个画面，每一个画面都用来描述一个不同的启动阶段。本文将详细分析这三个开机画面的显示过程，以便可以开启我们对Android系统UI实现的分析之路。

《Android系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

第一个开机画面是在内核启动的过程中出现的，它是一个静态的画面。第二个开机画面是在init进程启动的过程中出现的，它也是一个静态的画面。第三个开机画面是在系统服务启动的过程中出现的，它是一个动态的画面。无论是哪一个画面，它们都是在一个称为帧缓冲区（frame buffer，简称fb）的硬件设备上进行渲染的。接下来，我们就分别分析这三个画面是如何在fb上显示的。

## 第一个开机画面的显示过程

Android系统的第一个开机画面其实是[Linux](http://lib.csdn.net/base/linux)内核的启动画面。在默认情况下，这个画面是不会出现的，除非我们在编译内核的时候，启用以下两个编译选项：

**CONFIG_FRAMEBUFFER_CONSOLE**

**CONFIG_LOGO**

第一个编译选项表示内核支持帧缓冲区控制台，它对应的配置菜单项为**：Device Drivers ---> Graphics support ---> Console display driver support ---> Framebuffer Console support**。第二个编译选项表示内核在启动的过程中，需要显示LOGO，它对应的配置菜单项为：**Device Drivers ---> Graphics support ---> Bootup logo**。配置Android内核编译选项可以参考[在Ubuntu上下载、编译和安装Android最新内核源代码（Linux Kernel）](http://blog.csdn.net/luoshengyang/article/details/6564592)一文。

帧缓冲区硬件设备在内核中有一个对应的驱动程序模块fbmem，它实现在文件kernel/goldfish/drivers/video/fbmem.c中，它的初始化函数如下所示：

```c
/** 
*      fbmem_init - init frame buffer subsystem 
* 
*      Initialize the frame buffer subsystem. 
* 
*      NOTE: This function is _only_ to be called by drivers/char/mem.c. 
* 
*/  

static int __init  
fbmem_init(void)  
{  
proc_create("fb", 0, NULL, &fb_proc_fops);  

if (register_chrdev(FB_MAJOR,"fb",&fb_fops))  
        printk("unable to get major %d for fb devs\n", FB_MAJOR);  

fb_class = class_create(THIS_MODULE, "graphics");  
if (IS_ERR(fb_class)) {  
        printk(KERN_WARNING "Unable to create fb class; errno = %ld\n", PTR_ERR(fb_class));  
        fb_class = NULL;  
}  
return 0;  
}  
```
这个函数首先调用函数proc_create在/proc目录下创建了一个fb文件，接着又调用函数register_chrdev来注册了一个名称为fb的字符设备，最后调用函数class_create在/sys/class目录下创建了一个graphics目录，用来描述内核的图形系统。

模块fbmem除了会执行上述初始化工作之外，还会导出一个函数register_framebuffer：

```c
EXPORT_SYMBOL(register_framebuffer);  
```
这个函数在内核的启动过程会被调用，以便用来执行注册帧缓冲区硬件设备的操作，它的实现如下所示：

```c
/** 
*      register_framebuffer - registers a frame buffer device 
*      @fb_info: frame buffer info structure 
* 
*      Registers a frame buffer device @fb_info. 
* 
*      Returns negative errno on error, or zero for success. 
* 
*/  

int  
register_framebuffer(struct fb_info *fb_info)  
{  
int i;  
struct fb_event event;  
......  

if (num_registered_fb == FB_MAX)  
        return -ENXIO;  

......  

num_registered_fb++;  
for (i = 0 ; i < FB_MAX; i++)  
        if (!registered_fb[i])  
                break;  
fb_info->node = i;  
mutex_init(&fb_info->lock);  
fb_info->dev = device_create(fb_class, fb_info->device,  
                             MKDEV(FB_MAJOR, i), NULL, "fb%d", i);  
if (IS_ERR(fb_info->dev)) {  
        /* Not fatal */  
        printk(KERN_WARNING "Unable to create device for framebuffer %d; errno = %ld\n", i, PTR_ERR(fb_info->dev));  
        fb_info->dev = NULL;  
} else  
        fb_init_device(fb_info);  

......  

registered_fb[i] = fb_info;  

event.info = fb_info;  
fb_notifier_call_chain(FB_EVENT_FB_REGISTERED, &event);  
return 0;  
}  
```
由于系统中可能会存在多个帧缓冲区硬件设备，因此，fbmem模块使用一个数组registered_fb保存所有已经注册了的帧缓冲区硬件设备，其中，每一个帧缓冲区硬件都是使用一个结构体fb_info来描述的。

我们知道，在[linux](http://lib.csdn.net/base/linux)内核中，每一个硬件设备都有一个主设备号和一个从设备号，它们用来唯一地标识一个硬件设备。对于帧缓冲区硬件设备来说，它们的主设备号定义为FB_MAJOR（29），而从设备号则与注册的顺序有关，它们的值依次等于0，1，2等。

每一个被注册的帧缓冲区硬件设备在/dev/graphics目录下都有一个对应的设备文件fb<minor>，其中，<minor>表示一个从设备号。例如，第一个被注册的帧缓冲区硬件设备在/dev/graphics目录下都有一个对应的设备文件fb0。用户空间的应用程序通过这个设备文件就可以操作帧缓冲区硬件设备了，即将要显示的画面渲染到帧缓冲区硬件设备上去。

这个函数最后会通过调用函数fb_notifier_call_chain来通知帧缓冲区控制台，有一个新的帧缓冲区设备被注册到内核中来了。

帧缓冲区控制台在内核中对应的驱动程序模块为fbcon，它实现在文件kernel/goldfish/drivers/video/console/fbcon.c中，它的初始化函数如下所示：

```c
static struct notifier_block fbcon_event_notifier = {  
.notifier_call  = fbcon_event_notify,  
};  

......  

static int __init fb_console_init(void)  
{  
int i;  

acquire_console_sem();  
fb_register_client(&fbcon_event_notifier);  
fbcon_device = device_create(fb_class, NULL, MKDEV(0, 0), NULL,  
                             "fbcon");  

if (IS_ERR(fbcon_device)) {  
        printk(KERN_WARNING "Unable to create device "  
               "for fbcon; errno = %ld\n",  
               PTR_ERR(fbcon_device));  
        fbcon_device = NULL;  
} else  
        fbcon_init_device();  

for (i = 0; i < MAX_NR_CONSOLES; i++)  
        con2fb_map[i] = -1;  

release_console_sem();  
fbcon_start();  
return 0;  
}  
```
这个函数除了会调用函数device_create来创建一个类别为graphics的设备fbcon之外，还会调用函数fb_register_client来监听帧缓冲区硬件设备的注册事件，这是由函数fbcon_event_notify来实现的，如下所示：

```c
static int fbcon_event_notify(struct notifier_block *self,  
                      unsigned long action, void *data)  
{  
struct fb_event *event = data;  
struct fb_info *info = event->info;  
......  
int ret = 0;  

......  

switch(action) {  
......  
case FB_EVENT_FB_REGISTERED:  
        ret = fbcon_fb_registered(info);  
        break;  
......  

}  

done:  
return ret;  
}  
```
帧缓冲区硬件设备的注册事件最终是由函数fbcon_fb_registered来处理的，它的实现如下所示：

```c
static int fbcon_fb_registered(struct fb_info *info)  
{  
int ret = 0, i, idx = info->node;  

fbcon_select_primary(info);  

if (info_idx == -1) {  
        for (i = first_fb_vc; i <= last_fb_vc; i++) {  
                if (con2fb_map_boot[i] == idx) {  
                        info_idx = idx;  
                        break;  
                }  
        }  
    
        if (info_idx != -1)  
                ret = fbcon_takeover(1);  
} else {  
        for (i = first_fb_vc; i <= last_fb_vc; i++) {  
                if (con2fb_map_boot[i] == idx)  
                        set_con2fb_map(i, idx, 0);  
        }  
}  

return ret;  
}  
```
函数fbcon_select_primary用来检查当前注册的帧缓冲区硬件设备是否是一个主帧缓冲区硬件设备。如果是的话，那么就将它的信息记录下来。这个函数只有当指定了CONFIG_FRAMEBUFFER_CONSOLE_DETECT_PRIMARY编译选项时才有效，否则的话，它是一个空函数。

在Linux内核中，每一个控制台和每一个帧缓冲区硬件设备都有一个从0开始的编号，它们的初始对应关系保存在全局数组con2fb_map_boot中。控制台和帧缓冲区硬件设备的初始对应关系是可以通过设置内核启动参数来初始化的。在模块fbcon中，还有另外一个全局数组con2fb_map，也是用来映射控制台和帧缓冲区硬件设备的对应关系，不过它映射的是控制台和帧缓冲区硬件设备的实际对应关系。

全局变量first_fb_vc和last_fb_vc是全局数组con2fb_map_boot和con2fb_map的索引值，用来指定系统当前可用的控制台编号范围，它们也是可以通过设置内核启动参数来初始化的。全局变量first_fb_vc的默认值等于0，而全局变量last_fb_vc的默认值等于MAX_NR_CONSOLES - 1。

全局变量info_idx表示系统当前所使用的帧缓冲区硬件的编号。如果它的值等于-1，那么就说明系统当前还没有设置好当前所使用的帧缓冲区硬件设备。在这种情况下，函数fbcon_fb_registered就会在全局数组con2fb_map_boot中检查是否存在一个控制台编号与当前所注册的帧缓冲区硬件设备的编号idx对应。如果存在的话，那么就会将当前所注册的帧缓冲区硬件设备编号idx保存在全局变量info_idx中。接下来还会调用函数fbcon_takeover来初始化系统所使用的控制台。在调用函数fbcon_takeover的时候，传进去的参数为1，表示要显示第一个开机画面。

如果全局变量info_idx的值不等于-1，那么函数fbcon_fb_registered同样会在全局数组con2fb_map_boot中检查是否存在一个控制台编号与当前所注册的帧缓冲区硬件设备的编号idx对应。如果存在的话，那么就会调用函数set_con2fb_map来调整当前所注册的帧缓冲区硬件设备与控制台的映射关系，即调整数组con2fb_map_boot和con2fb_map的值。

为了简单起见，我们假设系统只有一个帧缓冲区硬件设备，这样当它被注册的时候，全局变量info_idx的值就会等于-1。当函数fbcon_fb_registered在全局数组con2fb_map_boot中发现有一个控制台的编号与这个帧缓冲区硬件设备的编号idx对应时，接下来就会调用函数fbcon_takeover来设置系统所使用的控制台。

函数fbcon_takeover的实现如下所示：

```c
static int fbcon_takeover(int show_logo)  
{  
int err, i;  

if (!num_registered_fb)  
        return -ENODEV;  

if (!show_logo)  
        logo_shown = FBCON_LOGO_DONTSHOW;  

for (i = first_fb_vc; i <= last_fb_vc; i++)  
        con2fb_map[i] = info_idx;  

err = take_over_console(&fb_con, first_fb_vc, last_fb_vc,  
                        fbcon_is_default);  

if (err) {  
        for (i = first_fb_vc; i <= last_fb_vc; i++) {  
                con2fb_map[i] = -1;  
        }  
        info_idx = -1;  
}  

return err;  
}  
```
全局变量logo_shown的初始值为FBCON_LOGO_CANSHOW，表示可以显示第一个开机画面。但是当参数show_logo的值等于0的时候，全局变量logo_shown的值会被重新设置为FBCON_LOGO_DONTSHOW，表示不可以显示第一个开机画面。

中间的for循环将当前可用的控制台的编号都映射到当前正在注册的帧缓冲区硬件设备的编号info_idx中去，表示当前可用的控制台与缓冲区硬件设备的实际映射关系。

函数take_over_console用来初始化系统当前所使用的控制台。如果它的返回值不等于0，那么就表示初始化失败。在这种情况下，最后的for循环就会将全局数组con2fb_map的各个元素的值设置为-1，表示系统当前可用的控制台还没有映射到实际的帧缓冲区硬件设备中去。这时候全局变量info_idx的值也会被重新设置为-1。

调用函数take_over_console来初始化系统当前所使用的控制台，实际上就是向系统注册一系列回调函数，以便系统可以通过这些回调函数来操作当前所使用的控制台。这些回调函数使用结构体consw来描述。这里所注册的结构体consw是由全局变量fb_con来指定的，它的定义如下所示：

```c
/* 
*  The console `switch' structure for the frame buffer based console 
   */  

static const struct consw fb_con = {  
.owner                  = THIS_MODULE,  
.con_startup            = fbcon_startup,  
.con_init               = fbcon_init,  
.con_deinit             = fbcon_deinit,  
.con_clear              = fbcon_clear,  
.con_putc               = fbcon_putc,  
.con_putcs              = fbcon_putcs,  
.con_cursor             = fbcon_cursor,  
.con_scroll             = fbcon_scroll,  
.con_bmove              = fbcon_bmove,  
.con_switch             = fbcon_switch,  
.con_blank              = fbcon_blank,  
.con_font_set           = fbcon_set_font,  
.con_font_get           = fbcon_get_font,  
.con_font_default       = fbcon_set_def_font,  
.con_font_copy          = fbcon_copy_font,  
.con_set_palette        = fbcon_set_palette,  
.con_scrolldelta        = fbcon_scrolldelta,  
.con_set_origin         = fbcon_set_origin,  
.con_invert_region      = fbcon_invert_region,  
.con_screen_pos         = fbcon_screen_pos,  
.con_getxy              = fbcon_getxy,  
.con_resize             = fbcon_resize,  
};  
```
接下来我们主要关注函数fbcon_init和fbcon_switch的实现，系统就是通过它来初始化和切换控制台的。在初始化的过程中，会决定是否需要准备第一个开机画面的内容，而在切换控制台的过程中，会决定是否需要显示第一个开机画面的内容。

函数fbcon_init的实现如下所示：

```c
static void fbcon_init(struct vc_data *vc, int init)  
{  
struct fb_info *info = registered_fb[con2fb_map[vc->vc_num]];  
struct fbcon_ops *ops;  
struct vc_data **default_mode = vc->vc_display_fg;  
struct vc_data *svc = *default_mode;  
struct display *t, *p = &fb_display[vc->vc_num];  
int logo = 1, new_rows, new_cols, rows, cols, charcnt = 256;  
int cap;  

if (info_idx == -1 || info == NULL)  
    return;  

......  

if (vc != svc || logo_shown == FBCON_LOGO_DONTSHOW ||  
    (info->fix.type == FB_TYPE_TEXT))  
        logo = 0;  

......  

if (logo)  
        fbcon_prepare_logo(vc, info, cols, rows, new_cols, new_rows);  

......  
}  
```
当前正在初始化的控制台使用参数vc来描述，而它的成员变量vc_num用来描述当前正在初始化的控制台的编号。通过这个编号之后，就可以在全局数组con2fb_map中找到对应的帧缓冲区硬件设备编号。有了帧缓冲区硬件设备编号之后，就可以在另外一个全局数组中registered_fb中找到一个fb_info结构体info，用来描述与当前正在初始化的控制台所对应的帧缓冲区硬件设备。

参数vc的成员变量vc_display_fg用来描述系统当前可见的控制台，它是一个类型为vc_data**的指针。从这里就可以看出，最终得到的vc_data结构体svc就是用来描述系统当前可见的控制台的。

变量logo开始的时候被设置为1，表示需要显示第一个开机画面，但是在以下三种情况下，它的值会被设置为0，表示不需要显示开机画面：

A. 参数vc和变量svc指向的不是同一个vc_data结构体，即当前正在初始化的控制台不是系统当前可见的控制台。

B. 全局变量logo_shown的值等于FBCON_LOGO_DONTSHOW，即系统不需要显示第一个开机画面。

C. 与当前正在初始化的控制台所对应的帧缓冲区硬件设备的显示方式被设置为文本方式，即info->fix.type的值等于FB_TYPE_TEXT。

当最终得到的变量logo的值等于1的时候，接下来就会调用函数fbcon_prepare_logo来准备要显示的第一个开机画面的内容。

在函数fbcon_prepare_logo中，第一个开机画面的内容是通过调用函数fb_prepare_logo来准备的，如下所示：

```c
static void fbcon_prepare_logo(struct vc_data *vc, struct fb_info *info,  
                       int cols, int rows, int new_cols, int new_rows)  
{  
......  

int logo_height;  

......  

logo_height = fb_prepare_logo(info, ops->rotate);  

......  

if (logo_lines > vc->vc_bottom) {  
        ......  
} else if (logo_shown != FBCON_LOGO_DONTSHOW) {  
        logo_shown = FBCON_LOGO_DRAW;  
        ......  
}  
}  
```
从函数fb_prepare_logo返回来之后，如果要显示的第一个开机画面所占用的控制台行数小于等于参数vc所描述的控制台的最大行数，并且全局变量logo_show的值不等于FBCON_LOGO_DONTSHOW，那么就说明前面所提到的第一个开机画面可以显示在控制台中。这时候全局变量logo_show的值就会被设置为FBCON_LOGO_DRAW，表示第一个开机画面处于等待渲染的状态。

函数fb_prepare_logo实现在文件kernel/goldfish/drivers/video/fbmem.c中，如下所示：

```c
int fb_prepare_logo(struct fb_info *info, int rotate)  
{  
int depth = fb_get_color_depth(&info->var, &info->fix);  
unsigned int yres;  

memset(&fb_logo, 0, sizeof(struct logo_data));  

......  

if (info->fix.visual == FB_VISUAL_DIRECTCOLOR) {  
        depth = info->var.blue.length;  
        if (info->var.red.length < depth)  
                depth = info->var.red.length;  
        if (info->var.green.length < depth)  
                depth = info->var.green.length;  
}  

if (info->fix.visual == FB_VISUAL_STATIC_PSEUDOCOLOR && depth > 4) {  
        /* assume console colormap */  
        depth = 4;  
}  

/* Return if no suitable logo was found */  
fb_logo.logo = fb_find_logo(depth);  

......  

return fb_prepare_extra_logos(info, fb_logo.logo->height, yres);  
}  
```
这个函数首先得到参数info所描述的帧缓冲区硬件设备的颜色深度depth，接着再调用函数fb_find_logo来获得要显示的第一个开机画面的内容，并且保存在全局变量fb_logo的成员变量logo中。

函数fb_find_logo实现在文件kernel/goldfish/drivers/video/logo/logo.c文件中，如下所示：

```c
extern const struct linux_logo logo_linux_mono;  
extern const struct linux_logo logo_linux_vga16;  
extern const struct linux_logo logo_linux_clut224;  
extern const struct linux_logo logo_blackfin_vga16;  
extern const struct linux_logo logo_blackfin_clut224;  
extern const struct linux_logo logo_dec_clut224;  
extern const struct linux_logo logo_mac_clut224;  
extern const struct linux_logo logo_parisc_clut224;  
extern const struct linux_logo logo_sgi_clut224;  
extern const struct linux_logo logo_sun_clut224;  
extern const struct linux_logo logo_superh_mono;  
extern const struct linux_logo logo_superh_vga16;  
extern const struct linux_logo logo_superh_clut224;  
extern const struct linux_logo logo_m32r_clut224;  

static int nologo;  
module_param(nologo, bool, 0);  
MODULE_PARM_DESC(nologo, "Disables startup logo");  

/* logo's are marked __initdata. Use __init_refok to tell 
* modpost that it is intended that this function uses data 
* marked __initdata. 
  */  
  const struct linux_logo * __init_refok fb_find_logo(int depth)  
  {  
  const struct linux_logo *logo = NULL;  

if (nologo)  
        return NULL;  

if (depth >= 1) {  
#ifdef CONFIG_LOGO_LINUX_MONO  
        /* Generic Linux logo */  
        logo = &logo_linux_mono;  
#endif  
#ifdef CONFIG_LOGO_SUPERH_MONO  
        /* SuperH Linux logo */  
        logo = &logo_superh_mono;  
#endif  
}  

if (depth >= 4) {  
#ifdef CONFIG_LOGO_LINUX_VGA16  
        /* Generic Linux logo */  
        logo = &logo_linux_vga16;  
#endif  
#ifdef CONFIG_LOGO_BLACKFIN_VGA16  
        /* Blackfin processor logo */  
        logo = &logo_blackfin_vga16;  
#endif  
#ifdef CONFIG_LOGO_SUPERH_VGA16  
        /* SuperH Linux logo */  
        logo = &logo_superh_vga16;  
#endif  
}  

if (depth >= 8) {  
#ifdef CONFIG_LOGO_LINUX_CLUT224  
        /* Generic Linux logo */  
        logo = &logo_linux_clut224;  
#endif  
#ifdef CONFIG_LOGO_BLACKFIN_CLUT224  
        /* Blackfin Linux logo */  
        logo = &logo_blackfin_clut224;  
#endif  
#ifdef CONFIG_LOGO_DEC_CLUT224  
        /* DEC Linux logo on MIPS/MIPS64 or ALPHA */  
        logo = &logo_dec_clut224;  
#endif  
#ifdef CONFIG_LOGO_MAC_CLUT224  
        /* Macintosh Linux logo on m68k */  
        if (MACH_IS_MAC)  
                logo = &logo_mac_clut224;  
#endif  
#ifdef CONFIG_LOGO_PARISC_CLUT224  
        /* PA-RISC Linux logo */  
        logo = &logo_parisc_clut224;  
#endif  
#ifdef CONFIG_LOGO_SGI_CLUT224  
        /* SGI Linux logo on MIPS/MIPS64 and VISWS */  
        logo = &logo_sgi_clut224;  
#endif  
#ifdef CONFIG_LOGO_SUN_CLUT224  
        /* Sun Linux logo */  
        logo = &logo_sun_clut224;  
#endif  
#ifdef CONFIG_LOGO_SUPERH_CLUT224  
        /* SuperH Linux logo */  
        logo = &logo_superh_clut224;  
#endif  
#ifdef CONFIG_LOGO_M32R_CLUT224  
        /* M32R Linux logo */  
        logo = &logo_m32r_clut224;  
#endif  
}  
return logo;  
}  
EXPORT_SYMBOL_GPL(fb_find_logo);  
```
文件开始声明的一系列linux_logo结构体变量分别用来保存kernel/goldfish/drivers/video/logo目录下的一系列ppm或者pbm文件的内容的。这些ppm或者pbm文件都是用来描述第一个开机画面的。

全局变量nologo是一个类型为布尔变量的模块参数，它的默认值等于0，表示要显示第一个开机画面。在这种情况下，函数fb_find_logo就会根据参数depth的值以及不同的编译选项来选择第一个开机画面的内容，并且保存在变量logo中返回给调用者。

这一步执行完成之后，第一个开机画面的内容就保存在模块fbmem的全局变量fb_logo的成员变量logo中了。这时候控制台的初始化过程也结束了，接下来系统就会执行切换控制台的操作。前面提到，当系统执行切换控制台的操作的时候，模块fbcon中的函数fbcon_switch就会被调用。在调用的过程中，就会执行显示第一个开机画面的操作。

函数fbcon_switch实现在文件kernel/goldfish/drivers/video/console/fbcon.c中，显示第一个开机画面的过程如下所示：

```c
static int fbcon_switch(struct vc_data *vc)  
{  
struct fb_info *info, *old_info = NULL;  
struct fbcon_ops *ops;  
struct display *p = &fb_display[vc->vc_num];  
struct fb_var_screeninfo var;  
int i, prev_console, charcnt = 256;  

......  

if (logo_shown == FBCON_LOGO_DRAW) {  
        logo_shown = fg_console;  
        /* This is protected above by initmem_freed */  
        fb_show_logo(info, ops->rotate);  
        ......  
        return 0;  
}  
return 1;  
}  
```
由于前面在准备第一个开机画面的内容的时候，全局变量logo_show的值被设置为FBCON_LOGO_DRAW，因此，接下来就会调用函数fb_show_logo来显示第一个开机画面。在显示之前，这个函数会将全局变量logo_shown的值设置为fg_console，后者表示系统当前可见的控制台的编号。

函数fb_show_logo实现在文件kernel/goldfish/drivers/video/fbmem.c中，如下所示：

```c
int fb_show_logo(struct fb_info *info, int rotate)  
{  
int y;  

y = fb_show_logo_line(info, rotate, fb_logo.logo, 0,  
                      num_online_cpus());  
......  

return y;  
}  
```
这个函数调用另外一个函数fb_show_logo_line来进一步执行渲染第一个开机画面的操作。

函数fb_show_logo_line也是实现在文件kernel/goldfish/drivers/video/fbmem.c中，如下所示：

```c
static int fb_show_logo_line(struct fb_info *info, int rotate,  
                     const struct linux_logo *logo, int y,  
                     unsigned int n)  
{  
u32 *palette = NULL, *saved_pseudo_palette = NULL;  
unsigned char *logo_new = NULL, *logo_rotate = NULL;  
struct fb_image image;  

/* Return if the frame buffer is not mapped or suspended */  
if (logo == NULL || info->state != FBINFO_STATE_RUNNING ||  
    info->flags & FBINFO_MODULE)  
        return 0;  

image.depth = 8;  
image.data = logo->data;  

if (fb_logo.needs_cmapreset)  
        fb_set_logocmap(info, logo);  

if (fb_logo.needs_truepalette ||  
    fb_logo.needs_directpalette) {  
        palette = kmalloc(256 * 4, GFP_KERNEL);  
        if (palette == NULL)  
                return 0;  

        if (fb_logo.needs_truepalette)  
                fb_set_logo_truepalette(info, logo, palette);  
        else  
                fb_set_logo_directpalette(info, logo, palette);  

        saved_pseudo_palette = info->pseudo_palette;  
        info->pseudo_palette = palette;  
}  

if (fb_logo.depth <= 4) {  
        logo_new = kmalloc(logo->width * logo->height, GFP_KERNEL);  
        if (logo_new == NULL) {  
                kfree(palette);  
                if (saved_pseudo_palette)  
                        info->pseudo_palette = saved_pseudo_palette;  
                return 0;  
        }  
        image.data = logo_new;  
        fb_set_logo(info, logo, logo_new, fb_logo.depth);  
}  

image.dx = 0;  
image.dy = y;  
image.width = logo->width;  
image.height = logo->height;  

if (rotate) {  
        logo_rotate = kmalloc(logo->width *  
                              logo->height, GFP_KERNEL);  
        if (logo_rotate)  
                fb_rotate_logo(info, logo_rotate, &image, rotate);  
}  

fb_do_show_logo(info, &image, rotate, n);  

kfree(palette);  
if (saved_pseudo_palette != NULL)  
        info->pseudo_palette = saved_pseudo_palette;  
kfree(logo_new);  
kfree(logo_rotate);  
return logo->height;  
}  
```
参数logo指向了前面所准备的第一个开机画面的内容。这个函数首先根据参数logo的内容来构造一个fb_image结构体image，用来描述最终要显示的第一个开机画面。最后就调用函数fb_do_show_logo来真正执行渲染第一个开机画面的操作。

函数fb_do_show_logo也是实现在文件kernel/goldfish/drivers/video/fbmem.c中，如下所示：

```c
static void fb_do_show_logo(struct fb_info *info, struct fb_image *image,  
                    int rotate, unsigned int num)  
{  
unsigned int x;  

if (rotate == FB_ROTATE_UR) {  
        for (x = 0;  
             x < num && image->dx + image->width <= info->var.xres;  
             x++) {  
                info->fbops->fb_imageblit(info, image);  
                image->dx += image->width + 8;  
        }  
} else if (rotate == FB_ROTATE_UD) {  
        for (x = 0; x < num && image->dx >= 0; x++) {  
                info->fbops->fb_imageblit(info, image);  
                image->dx -= image->width + 8;  
        }  
} else if (rotate == FB_ROTATE_CW) {  
        for (x = 0;  
             x < num && image->dy + image->height <= info->var.yres;  
             x++) {  
                info->fbops->fb_imageblit(info, image);  
                image->dy += image->height + 8;  
        }  
} else if (rotate == FB_ROTATE_CCW) {  
        for (x = 0; x < num && image->dy >= 0; x++) {  
                info->fbops->fb_imageblit(info, image);  
                image->dy -= image->height + 8;  
        }  
}  
}  
```
参数rotate用来描述屏幕的当前旋转方向。屏幕旋转方向不同，第一个开机画面的渲染方式也有所不同。例如，当屏幕上下颠倒时（FB_ROTATE_UD），第一个开机画面的左右顺序就刚好调换过来，这时候就需要从右到左来渲染。其它三个方向FB_ROTATE_UR、FB_ROTATE_CW和FB_ROTATE_CCW分别表示没有旋转、顺时针旋转90度和逆时针旋转90度。

参数info用来描述要渲染的帧缓冲区硬件设备，它的成员变量fbops指向了一系列回调函数，用来操作帧缓冲区硬件设备，其中，回调函数fb_imageblit就是用来在指定的帧缓冲区硬件设备渲染指定的图像的。

至此，第一个开机画面的显示过程就分析完成了。

## 第二个开机画面的显示过程

由于第二个开机画面是在init进程启动的过程中显示的，因此，我们就从init进程的入口函数main开始分析第二个开机画面的显示过程。

init进程的入口函数main实现在文件system/core/init/init.c中，如下所示：

```c
int main(int argc, char **argv)  
{  
    int fd_count = 0;  
    struct pollfd ufds[4];  
    ......  
    int property_set_fd_init = 0;  
    int signal_fd_init = 0;  
    int keychord_fd_init = 0;  

    if (!strcmp(basename(argv[0]), "ueventd"))  
return ueventd_main(argc, argv);  

    ......  

    queue_builtin_action(console_init_action, "console_init");  

    ......  

    for(;;) {  
int nr, i, timeout = -1;  

execute_one_command();  
restart_processes();  

if (!property_set_fd_init && get_property_set_fd() > 0) {  
    ufds[fd_count].fd = get_property_set_fd();  
    ufds[fd_count].events = POLLIN;  
    ufds[fd_count].revents = 0;  
    fd_count++;  
    property_set_fd_init = 1;  
}  
if (!signal_fd_init && get_signal_fd() > 0) {  
    ufds[fd_count].fd = get_signal_fd();  
    ufds[fd_count].events = POLLIN;  
    ufds[fd_count].revents = 0;  
    fd_count++;  
    signal_fd_init = 1;  
}  
if (!keychord_fd_init && get_keychord_fd() > 0) {  
    ufds[fd_count].fd = get_keychord_fd();  
    ufds[fd_count].events = POLLIN;  
    ufds[fd_count].revents = 0;  
    fd_count++;  
    keychord_fd_init = 1;  
}  

if (process_needs_restart) {  
    timeout = (process_needs_restart - gettime()) * 1000;  
    if (timeout < 0)  
        timeout = 0;  
}  

if (!action_queue_empty() || cur_action)  
    timeout = 0;  

......  

nr = poll(ufds, fd_count, timeout);  
if (nr <= 0)  
    continue;  

for (i = 0; i < fd_count; i++) {  
    if (ufds[i].revents == POLLIN) {  
        if (ufds[i].fd == get_property_set_fd())  
            handle_property_set_fd();  
        else if (ufds[i].fd == get_keychord_fd())  
            handle_keychord();  
        else if (ufds[i].fd == get_signal_fd())  
            handle_signal();  
    }  
}  
    }  

    return 0;  
}  
```
 函数一开始就首先判断参数argv[0]的值是否等于“ueventd”，即当前正在启动的进程名称是否等于“ueventd”。如果是的话，那么就以ueventd_main函数来作入口函数。这是怎么回事呢？当前正在启动的进程不是init吗？它的名称怎么可能会等于“ueventd”？原来，在目标设备上，可执行文件/sbin/ueventd是可执行文件/init的一个符号链接文件，即应用程序ueventd和init运行的是同一个可执行文件。内核启动完成之后，可执行文件/init首先会被执行，即init进程会首先被启动。init进程在启动的过程中，会对启动脚本/init.rc进行解析。在启动脚本/init.rc中，配置了一个ueventd进程，它对应的可执行文件为/sbin/ueventd，即ueventd进程加载的可执行文件也为/init。因此，通过判断参数argv[0]的值，就可以知道当前正在启动的是init进程还是ueventd进程。

ueventd进程是作什么用的呢？它是用来处理uevent事件的，即用来管理系统设备的。从前面的描述可以知道，它真正的入口函数为ueventd_main，实现在system/core/init/ueventd.c中。ueventd进程会通过一个socket接口来和内核通信，以便可以监控系统设备事件。例如，在前面[在Ubuntu上为Android系统编写Linux内核驱动程序](http://blog.csdn.net/luoshengyang/article/details/6568411)一文中， 我们调用device_create函数来创建了一个名称为“hello”的字符设备，这时候内核就会向前面提到的socket发送一个设备增加事件。ueventd进程通过这个socket获得了这个设备增加事件之后，就会/dev目录下创建一个名称为“hello”的设备文件。这样用户空间的应用程序就可以通过设备文件/dev/hello来和驱动程序hello进行通信了。

接下来调用另外一个函数queue_builtin_action来向init进程中的一个待执行action队列增加了一个名称等于“console_init”的action。这个action对应的执行函数为console_init_action，它就是用来显示第二个开机画面的。

函数queue_builtin_action实现在文件system/core/init/init_parser.c文件中，如下所示：

```c
static list_declare(action_list);  
static list_declare(action_queue);  

void queue_builtin_action(int (*func)(int nargs, char **args), char *name)  
{  
    struct action *act;  
    struct command *cmd;  
    
    act = calloc(1, sizeof(*act));  
    act->name = name;  
    list_init(&act->commands);  
    
    cmd = calloc(1, sizeof(*cmd));  
    cmd->func = func;  
    cmd->args[0] = name;  
    list_add_tail(&act->commands, &cmd->clist);  
    
    list_add_tail(&action_list, &act->alist);  
    action_add_queue_tail(act);  
}  

void action_add_queue_tail(struct action *act)  
{  
    list_add_tail(&action_queue, &act->qlist);  
}  
```
action_list列表用来保存从启动脚本/init.rc解析得到的一系列action，以及一系列内建的action。当这些action需要执行的时候，它们就会被添加到action_queue列表中去，以便init进程可以执行它们。

回到init进程的入口函数main中，最后init进程会进入到一个无限循环中去。在这个无限循环中，init进程会做以下五个事情：

A. 调用函数execute_one_command来检查action_queue列表是否为空。如果不为空的话，那么init进程就会将保存在列表头中的action移除，并且执行这个被移除的action。由于前面我们将一个名称为“console_init”的action添加到了action_queue列表中，因此，在这个无限循环中，这个action就会被执行，即函数console_init_action会被调用。

B. 调用函数restart_processes来检查系统中是否有进程需要重启。在启动脚本/init.rc中，我们可以指定一个进程在退出之后会自动重新启动。在这种情况下，函数restart_processes就会检查是否存在需要重新启动的进程，如果存在的话，那么就会将它重新启动起来。

C. 处理系统属性变化事件。当我们调用函数property_set来改变一个系统属性值时，系统就会通过一个socket（通过调用函数get_property_set_fd可以获得它的文件描述符）来向init进程发送一个属性值改变事件通知。init进程接收到这个属性值改变事件之后，就会调用函数handle_property_set_fd来进行相应的处理。后面在分析第三个开机画面的显示过程时，我们就会看到，SurfaceFlinger服务就是通过修改“ctl.start”和“ctl.stop”属性值来启动和停止第三个开机画面的。

D. 处理一种称为“chorded keyboard”的键盘输入事件。这种类型为chorded keyboard”的键盘设备通过不同的铵键组合来描述不同的命令或者操作，它对应的设备文件为/dev/keychord。我们可以通过调用函数get_keychord_fd来获得这个设备的文件描述符，以便可以监控它的输入事件，并且调用函数handle_keychord来对这些输入事件进行处理。

E. 回收僵尸进程。我们知道，在Linux内核中，如果父进程不等待子进程结束就退出，那么当子进程结束的时候，就会变成一个僵尸进程，从而占用系统的资源。为了回收这些僵尸进程，init进程会安装一个SIGCHLD信号接收器。当那些父进程已经退出了的子进程退出的时候，内核就会发出一个SIGCHLD信号给init进程。init进程可以通过一个socket（通过调用函数get_signal_fd可以获得它的文件描述符）来将接收到的SIGCHLD信号读取回来，并且调用函数handle_signal来对接收到的SIGCHLD信号进行处理，即回收那些已经变成了僵尸的子进程。

注意，由于后面三个事件都是可以通过文件描述符来描述的，因此，init进程的入口函数main使用poll机制来同时轮询它们，以便可以提高效率。

接下来我们就重点分析函数console_init_action的实现，以便可以了解第二个开机画面的显示过程：

```c
static int console_init_action(int nargs, char **args)  
{  
    int fd;  
    char tmp[PROP_VALUE_MAX];  

    if (console[0]) {  
snprintf(tmp, sizeof(tmp), "/dev/%s", console);  
console_name = strdup(tmp);  
    }  

    fd = open(console_name, O_RDWR);  
    if (fd >= 0)  
have_console = 1;  
    close(fd);  

    if( load_565rle_image(INIT_IMAGE_FILE) ) {  
fd = open("/dev/tty0", O_WRONLY);  
if (fd >= 0) {  
    const char *msg;  
        msg = "\n"  
    "\n"  
    "\n"  
    "\n"  
    "\n"  
    "\n"  
    "\n"  // console is 40 cols x 30 lines  
    "\n"  
    "\n"  
    "\n"  
    "\n"  
    "\n"  
    "\n"  
    "\n"  
    "             A N D R O I D ";  
    write(fd, msg, strlen(msg));  
    close(fd);  
}  
    }  
    return 0;  
}  
```
这个函数主要做了两件事件：

A. 初始化控制台。init进程在启动的时候，会解析内核的启动参数（保存在文件/proc/cmdline中）。如果发现内核的启动参数中包含有了一个名称为“androidboot.console”的属性，那么就会将这个属性的值保存在字符数组console中。这样我们就可以通过设备文件/dev/<console>来访问系统的控制台。如果内核的启动参数没有包含名称为“androidboot.console”的属性，那么默认就通过设备文件/dev/console来访问系统的控制台。如果能够成功地打开设备文件/dev/<console>或者/dev/console，那么就说明系统支持访问控制台，因此，全局变量have_console的就会被设置为1。

B. 显示第二个开机画面。显示第二个开机画面是通过调用函数load_565rle_image来实现的。在调用函数load_565rle_image的时候，指定的开机画面文件为INIT_IMAGE_FILE。INIT_IMAGE_FILE是一个宏，定义在system/core/init/init.h文件中，如下所示：

```c
#define INIT_IMAGE_FILE "/initlogo.rle"  
```
即第二个开机画面的内容是由文件/initlogo.rle来指定的。如果文件/initlogo.rle不存在，或者在显示它的过程中出现异常，那么函数load_565rle_image的返回值就会等于-1，这时候函数console_init_action就以文本的方式来显示第二个开机画面，即向编号为0的控制台（/dev/tty0）输出“ANDROID”这7个字符。

函数load_565rle_image实现在文件system/core/init/logo.c中，如下所示：

```c
/* 565RLE image format: [count(2 bytes), rle(2 bytes)] */  

int load_565rle_image(char *fn)  
{  
    struct FB fb;  
    struct stat s;  
    unsigned short *data, *bits, *ptr;  
    unsigned count, max;  
    int fd;  

    if (vt_set_mode(1))  
return -1;  

    fd = open(fn, O_RDONLY);  
    if (fd < 0) {  
ERROR("cannot open '%s'\n", fn);  
goto fail_restore_text;  
    }  

    if (fstat(fd, &s) < 0) {  
goto fail_close_file;  
    }  

    data = mmap(0, s.st_size, PROT_READ, MAP_SHARED, fd, 0);  
    if (data == MAP_FAILED)  
goto fail_close_file;  

    if (fb_open(&fb))  
goto fail_unmap_data;  

    max = fb_width(&fb) * fb_height(&fb);  
    ptr = data;  
    count = s.st_size;  
    bits = fb.bits;  
    while (count > 3) {  
unsigned n = ptr[0];  
if (n > max)  
    break;  
android_memset16(bits, ptr[1], n << 1);  
bits += n;  
max -= n;  
ptr += 2;  
count -= 4;  
    }  

    munmap(data, s.st_size);  
    fb_update(&fb);  
    fb_close(&fb);  
    close(fd);  
    unlink(fn);  
    return 0;  

fail_unmap_data:  
    munmap(data, s.st_size);  
fail_close_file:  
    close(fd);  
fail_restore_text:  
    vt_set_mode(0);  
    return -1;  
}  
```
函数首先将控制台的显示方式设置为图形方式，这是通过调用函数vt_set_mode来实现的，如下所示：

```c
static int vt_set_mode(int graphics)  
{  
    int fd, r;  
    fd = open("/dev/tty0", O_RDWR | O_SYNC);  
    if (fd < 0)  
return -1;  
    r = ioctl(fd, KDSETMODE, (void*) (graphics ? KD_GRAPHICS : KD_TEXT));  
    close(fd);  
    return r;  
}  
```
函数vt_set_mode首先打开控制台设备文件/dev/tty0，接着再通过IO控制命令KDSETMODE来将控制台的显示方式设置为文本方式或者图形方式，取决于参数graphics的值。从前面的调用过程可以知道，参数graphics的值等于1，因此，这里是将控制台的显示方式设备为图形方式。

回到函数load_565rle_image中，从前面的调用过程可以知道，参数fn的值等于“/initlogo.rle”，即指向目标设备上的initlogo.rle文件。函数load_565rle_image首先调用函数open打开这个文件，并且将获得的文件描述符保存在变量fd中，接着再调用函数fstat来获得这个文件的大小。有了这些信息之后，函数load_565rle_image就可以调用函数mmap来把文件/initlogo.rle映射到init进程的地址空间来了，以便可以读取它的内容。

将文件/initlogo.rle映射到init进程的地址空间之后，接下来再调用函数fb_open来打开设备文件/dev/graphics/fb0。前面在介绍第一个开机画面的显示过程中提到，设备文件/dev/graphics/fb0是用来访问系统的帧缓冲区硬件设备的，因此，打开了设备文件/dev/graphics/fb0之后，我们就可以将文件/initlogo.rle的内容输出到帧缓冲区硬件设备中去了。

函数fb_open的实现如下所示：

```c
static int fb_open(struct FB *fb)  
{  
    fb->fd = open("/dev/graphics/fb0", O_RDWR);  
    if (fb->fd < 0)  
return -1;  

    if (ioctl(fb->fd, FBIOGET_FSCREENINFO, &fb->fi) < 0)  
goto fail;  
    if (ioctl(fb->fd, FBIOGET_VSCREENINFO, &fb->vi) < 0)  
goto fail;  

    fb->bits = mmap(0, fb_size(fb), PROT_READ | PROT_WRITE,  
            MAP_SHARED, fb->fd, 0);  
    if (fb->bits == MAP_FAILED)  
goto fail;  

    return 0;  

fail:  
    close(fb->fd);  
    return -1;  
}  
```
打开了设备文件/dev/graphics/fb0之后，接着再分别通过IO控制命令FBIOGET_FSCREENINFO和FBIOGET_VSCREENINFO来获得帧缓冲硬件设备的固定信息和可变信息。固定信息使用一个fb_fix_screeninfo结构体来描述，它保存的是帧缓冲区硬件设备固有的特性，这些特性在帧缓冲区硬件设备被初始化了之后，就不会发生改变，例如屏幕大小以及物理地址等信息。可变信息使用一个fb_var_screeninfo结构体来描述，它保存的是帧缓冲区硬件设备可变的特性，这些特性在系统运行的期间是可以改变的，例如屏幕所使用的分辨率、颜色深度以及颜色格式等。

除了获得帧缓冲区硬件设备的固定信息和可变信息之外，函数fb_open还会将设备文件/dev/graphics/fb0的内容映射到init进程的地址空间来，这样init进程就可以通过映射得到的虚拟地址来访问帧缓冲区硬件设备的内容了。

回到函数load_565rle_image中，接下来分别使用宏fb_width和fb_height来获得屏幕所使用的的分辨率，即屏幕的宽度和高度。宏fb_width和fb_height的定义如下所示：

```c
#define fb_width(fb) ((fb)->vi.xres)  
#define fb_height(fb) ((fb)->vi.yres)  
```
屏幕的所使用的分辨率使用结构体fb_var_screeninfo的成员变量xres和yres来描述，其中，成员变量xres用来描述屏幕的宽度，而成员变量成员变量yres用来描述屏幕的高度。得到了屏幕的分辨率之后，就可以知道最多可以向帧缓冲区硬件设备写入的字节数的大小了，这个大小就等于屏幕的宽度乘以高度，保存在变量max中。

现在我们分别得到了文件initlogo.rle和帧缓冲区硬件设备在init进程中的虚拟访问地址以及大小，这样我们就可以将文件initlogo.rle的内容写入到帧缓冲区硬件设备中去，以便可以将第二个开机画面显示出来，这是通过函数load_565rle_image中的while循环来实现的。

文件initlogo.rle保存的第二个开机画面的图像格式是565rle的。rle的全称是run-length encoding，翻译为游程编码或者行程长度编码，它可以使用4个字节来描述一个连续的具有相同颜色值的序列。在rle565格式，前面2个字节中用来描述序列的个数，而后面2个字节用来描述一个具体的颜色，其中，颜色的RGB值分别占5位、6位和5位。理解了565rle图像格式之后，我们就可以理解函数load_565rle_image中的while循环的实现逻辑了。在每一次循环中，都会依次从文件initlogo.rle中读出4个字节，其中，前两个字节的内容保存在变量n中，而后面2个字节的内容用来写入到帧缓冲区硬件设备中去。由于2个字节刚好就可以使用一个无符号短整数来描述，因此，函数load_565rle_image通过调用函数android_memset16来将从文件initlogo.rle中读取出来的颜色值写入到帧缓冲区硬件设备中去，

函数android_memset16的实现如下所示：

```c
void android_memset16(void *_ptr, unsigned short val, unsigned count)  
{  
    unsigned short *ptr = _ptr;  
    count >>= 1;  
    while(count--)  
*ptr++ = val;  
}  
```
参数ptr指向被写入的地址，在我们这个场景中，这个地址即为帧缓冲区硬件设备映射到init进程中的虚拟地址值。

参数val用来描述被写入的值，在我们这个场景中，这个值即为从文件initlogo.rle中读取出来的颜色值。

参数count用来描述被写入的地址的长度，它是以字节为单位的。由于在将参数val的值写入到参数ptr所描述的地址中去时，是以无符号短整数为单位的，即是以2个字节为单位的，因此，函数android_memset16在将参数val写入到地址ptr中去之前，首先会将参数count的值除以2。相应的地，在函数load_565rle_image中，需要将具有相同颜色值的序列的个数乘以2之后，再调用函数android_memset16。

回到函数load_565rle_image中，将文件/initlogo.rle的内容写入到帧缓冲区硬件设备去之后，第二个开机画面就可以显示出来了。接下来函数load_565rle_image就会调用函数munmap来注销文件/initlogo.rle在init进程中的映射，并且调用函数close来关闭文件/initlogo.rle。关闭了文件/initlogo.rle之后，还会调用函数unlink来删除目标设备上的/initlogo.rle文件。注意，这只是删除了目标设备上的/initlogo.rle文件，而不是删除ramdisk映像中的initlogo.rle文件，因此，每次关机启动之后，系统都会重新将ramdisk映像中的initlogo.rle文件安装到目标设备上的根目录来，这样就可以在每次开机的时候都能将它显示出来。

除了需要注销文件/initlogo.rle在init进程中的映射和关闭文件/initlogo.rle之外，还需要注销文件/dev/graphics/fb0在init进程中的映射以及关闭文件/dev/graphics/fb0，这是通过调用fb_close函数来实现的，如下所示：

```c
static void fb_close(struct FB *fb)  
{  
    munmap(fb->bits, fb_size(fb));  
    close(fb->fd);  
}  
```
在调用fb_close函数之前，函数load_565rle_image还会调用另外一个函数fb_update来更新屏幕上的第二个开机画面，它的实现如下所示：

```c
static void fb_update(struct FB *fb)  
{  
    fb->vi.yoffset = 1;  
    ioctl(fb->fd, FBIOPUT_VSCREENINFO, &fb->vi);  
    fb->vi.yoffset = 0;  
    ioctl(fb->fd, FBIOPUT_VSCREENINFO, &fb->vi);  
}  
```
在结构体fb_var_screeninfo中，除了使用成员变量xres和yres来描述屏幕所使用的分辨率之外，还使用成员变量xres_virtual和yres_virtual来描述屏幕所使用的虚拟分辨率。成员变量xres和yres所描述屏幕的分辨率称为可视分辨率。可视分辨率和虚拟分辨率有什么关系呢？可视分辨率是屏幕实际上使用的分辨率，即用户所看到的分辨率，而虚拟分辨率是在系统内部使用的，它是不可见的，并且可以大于可视分辨率。例如，假设可视分辨率是800 x 600，那么虚拟分辨率可以设置为1600 x 600。由于屏幕最多只可以显示800 x 600个像素，因此，在系统内部，就需要决定从1600 x 600中取出800 x 600个像素来显示，这是通过结构体fb_var_screeninfo的成员变量xoffset和yoffset的值来描述的。成员变量xoffset和yoffset的默认值等于0，即默认从虚拟分辨率的左上角取出与可视分辨率大小相等的像素出来显示，否则的话，就会根据成员变量xoffset和yoffset的值来从虚拟分辨率的中间位置取出与可视分辨率大小相等的像素出来显示。

帧缓冲区的大小是由虚拟分辨率决定的，因此，我们就可以在帧缓冲中写入比屏幕大小还要多的像素值，多出来的这个部分像素值就可以用作双缓冲。我们仍然假设可视分辨率和虚拟分辨率分别是800 x 600和1600 x 600，那么我们就可以先将前一个图像的内容写入到帧缓冲区的前面800 x 600个像素中去，接着再将后一个图像的内容写入到帧缓冲区的后面800 x 600个像素中。通过分别将用来描述帧缓冲区硬件设备的fb_var_screeninfo结构体的成员变量yoffset的值设置为0和800，就可以平滑地显示两个图像。

理解了帧缓冲区硬件设备的可视分辨性和虚拟分辨性之后，函数fb_update的实现逻辑就可以很好地理解了。
至此，第二个开机画面的显示过程就分析完成了。

## 第三个开机画面的显示过程

第三个开机画面是由应用程序bootanimation来负责显示的。应用程序bootanimation在启动脚本init.rc中被配置成了一个服务，如下所示：

```
service bootanim /system/bin/bootanimation  
    user graphics  
    group graphics  
    disabled  
    oneshot  
```
应用程序bootanimation的用户和用户组名称分别被设置为graphics。注意， 用来启动应用程序bootanimation的服务是disable的，即init进程在启动的时候，不会主动将应用程序bootanimation启动起来。当SurfaceFlinger服务启动的时候，它会通过修改系统属性ctl.start的值来通知init进程启动应用程序bootanimation，以便可以显示第三个开机画面，而当System进程将系统中的关键服务都启动起来之后，ActivityManagerService服务就会通知SurfaceFlinger服务来修改系统属性ctl.stop的值，以便可以通知init进程停止执行应用程序bootanimation，即停止显示第三个开机画面。接下来我们就分别分析第三个开机画面的显示过程和停止过程。

从前面[Android系统进程Zygote启动过程的源代码分析](http://blog.csdn.net/luoshengyang/article/details/6768304)一文可以知道，Zygote进程在启动的过程中，会将System进程启动起来，而从前面[Android应用程序安装过程源代码分析一文](http://blog.csdn.net/luoshengyang/article/details/6766010)又可以知道，System进程在启动的过程（Step 3）中，会调用SurfaceFlinger类的静态成员函数instantiate来启动SurfaceFlinger服务。Sytem进程在启动SurfaceFlinger服务的过程中，首先会创建一个SurfaceFlinger实例，然后再将这个实例注册到Service Manager中去。在注册的过程，前面创建的SurfaceFlinger实例会被一个sp指针引用。从前面[Android系统的智能指针（轻量级指针、强指针和弱指针）的实现原理分析一文](http://blog.csdn.net/luoshengyang/article/details/6786239)可以知道，当一个对象第一次被[智能](http://lib.csdn.net/base/aiplanning)指针引用的时候，这个对象的成员函数onFirstRef就会被调用。由于SurfaceFlinger重写了父类RefBase的成员函数onFirstRef，因此，在注册SurfaceFlinger服务的过程中，将会调用SurfaceFlinger类的成员函数onFirstRef。在调用的过程，就会创建一个线程来启动第三个开机画面。

SurfaceFlinger类实现在文件frameworks/base/services/surfaceflinger/SurfaceFlinger.cpp 中，它的成员函数onFirstRef的实现如下所示：

```c
void SurfaceFlinger::onFirstRef()  
{  
    run("SurfaceFlinger", PRIORITY_URGENT_DISPLAY);  

    // Wait for the main thread to be done with its initialization  
    mReadyToRunBarrier.wait();  
}  
```
SurfaceFlinger类继承了Thread类，当它的成员函数run被调用的时候，系统就会创建一个新的线程。这个线程在第一次运行之前，会调用SurfaceFlinger类的成员函数readyToRun来通知SurfaceFlinger，它准备就绪了。当这个线程准备就绪之后，它就会循环执行SurfaceFlinger类的成员函数threadLoop，直到这个成员函数的返回值等于false为止。

注意，SurfaceFlinger类的成员函数onFirstRef是在System进程的主线程中调用的，它需要等待前面创建的线程准备就绪之后，再继续往前执行，这个通过调用SurfaceFlinger类的成员变量mReadytoRunBarrier所描述的一个Barrier对象的成员函数wait来实现的。每一个Barrier对象内问都封装了一个条件变量（Condition Variable），而条件变量是用来同步线程的。

接下来，我们继续分析SurfaceFlinger类的成员函数readyToRun的实现，如下所示：

```c
status_t SurfaceFlinger::readyToRun()  
{  
    LOGI(   "SurfaceFlinger's main thread ready to run. "  
    "Initializing graphics H/W...");  
      
    ......  

    mReadyToRunBarrier.open();  

    /* 
     *  We're now ready to accept clients... 
     */  

    // start boot animation  
    property_set("ctl.start", "bootanim");  

    return NO_ERROR;  
}  
```
前面创建的线程用作SurfaceFlinger的主线程。这个线程在启动的时候，会对设备主屏幕以及OpenGL库进行初始化。初始化完成之后，接着就会调用SurfaceFlinger类的成员变量mReadyToRunBarrier所描述的一个Barrier对象的成员函数open来唤醒System进程的主线程，以便它可以继续往前执行。最后，SurfaceFlinger类的成员函数readyToRun的成员函数会调用函数property_set来将系统属性“ctl.start”的值设置为“bootanim”，表示要将应用程序bootanimation启动起来，以便可以显示第三个开机画面。

前面在介绍第二个开机画面的时候提到，当系统属性发生改变时，init进程就会接收到一个系统属性变化通知，这个通知最终是由在init进程中的函数handle_property_set_fd来处理的。

函数handle_property_set_fd实现在文件system/core/init/property_service.c中，如下所示：

```c
void handle_property_set_fd()  
{  
    prop_msg msg;  
    int s;  
    int r;  
    int res;  
    struct ucred cr;  
    struct sockaddr_un addr;  
    socklen_t addr_size = sizeof(addr);  
    socklen_t cr_size = sizeof(cr);  
    
    if ((s = accept(property_set_fd, (struct sockaddr *) &addr, &addr_size)) < 0) {  
return;  
    }  

    /* Check socket options here */  
    if (getsockopt(s, SOL_SOCKET, SO_PEERCRED, &cr, &cr_size) < 0) {  
close(s);  
ERROR("Unable to recieve socket options\n");  
return;  
    }  

    r = recv(s, &msg, sizeof(msg), 0);  
    close(s);  
    if(r != sizeof(prop_msg)) {  
ERROR("sys_prop: mis-match msg size recieved: %d expected: %d\n",  
      r, sizeof(prop_msg));  
return;  
    }  

    switch(msg.cmd) {  
    case PROP_MSG_SETPROP:  
msg.name[PROP_NAME_MAX-1] = 0;  
msg.value[PROP_VALUE_MAX-1] = 0;  

if(memcmp(msg.name,"ctl.",4) == 0) {  
    if (check_control_perms(msg.value, cr.uid, cr.gid)) {  
        handle_control_message((char*) msg.name + 4, (char*) msg.value);  
    } else {  
        ERROR("sys_prop: Unable to %s service ctl [%s] uid: %d pid:%d\n",  
                msg.name + 4, msg.value, cr.uid, cr.pid);  
    }  
} else {  
    if (check_perms(msg.name, cr.uid, cr.gid)) {  
        property_set((char*) msg.name, (char*) msg.value);  
    } else {  
        ERROR("sys_prop: permission denied uid:%d  name:%s\n",  
              cr.uid, msg.name);  
    }  
}  
break;  

    default:  
break;  
    }  
}  
```
init进程是通过一个socket来接收系统属性变化事件的。每一个系统属性变化事件的内容都是通过一个prop_msg对象来描述的。在prop_msg对象对，成员变量name用来描述发生变化的系统属性的名称，而成员变量value用来描述发生变化的系统属性的值。系统属性分为两种类型，一种是普通类型的系统属性，另一种是控制类型的系统属性（属性名称以“ctl.”开头）。控制类型的系统属性在发生变化时，会触发init进程执行一个命令，而普通类型的系统属性就不具有这个特性。注意，改变系统属性是需要权限，因此，函数handle_property_set_fd在处理一个系统属性变化事件之前，首先会检查修改系统属性的进程是否具有相应的权限，这是通过调用函数check_control_perms或者check_perms来实现的。

从前面的调用过程可以知道，当前发生变化的系统属性的名称为“ctl.start”，它的值被设置为“bootanim”。由于这是一个控制类型的系统属性，因此，在通过了权限检查之后，另外一个函数handle_control_message就会被调用，以便可以执行一个名称为“bootanim”的命令。

函数handle_control_message实现在system/core/init/init.c中，如下所示：

```c
void handle_control_message(const char *msg, const char *arg)  
{  
    if (!strcmp(msg,"start")) {  
msg_start(arg);  
    } else if (!strcmp(msg,"stop")) {  
msg_stop(arg);  
    } else {  
ERROR("unknown control msg '%s'\n", msg);  
    }  
}  
```
控制类型的系统属性的名称是以"ctl."开头，并且是以“start”或者“stop”结尾的，其中，“start”表示要启动某一个服务，而“stop”表示要停止某一个服务，它们是分别通过函数msg_start和msg_stop来实现的。由于当前发生变化的系统属性是以“start”来结尾的，因此，接下来就会调用函数msg_start来启动一个名称为“bootanim”的服务。 

函数msg_start实现在文件system/core/init/init.c中，如下所示：

```c
static void msg_start(const char *name)  
{  
    struct service *svc;  
    char *tmp = NULL;  
    char *args = NULL;  
    
    if (!strchr(name, ':'))  
svc = service_find_by_name(name);  
    else {  
tmp = strdup(name);  
args = strchr(tmp, ':');  
*args = '\0';  
args++;  

svc = service_find_by_name(tmp);  
    }  

    if (svc) {  
service_start(svc, args);  
    } else {  
ERROR("no such service '%s'\n", name);  
    }  
    if (tmp)  
free(tmp);  
}  
```
参数name的值等于“bootanim”，它用来描述一个服务名称。这个函数首先调用函数service_find_by_name来找到名称等于“bootanim”的服务的信息，这些信息保存在一个service结构体svc中，接着再调用另外一个函数service_start来将对应的应用程序启动起来。

从前面的内容可以知道，名称等于“bootanim”的服务所对应的应用程序为/system/bin/bootanimation，这个应用程序实现在frameworks/base/cmds/bootanimation目录中，其中，应用程序入口函数main是实现在frameworks/base/cmds/bootanimation/bootanimation_main.cpp中的，如下所示：

```c
int main(int argc, char** argv)  
{  
#if defined(HAVE_PTHREADS)  
    setpriority(PRIO_PROCESS, 0, ANDROID_PRIORITY_DISPLAY);  
#endif  

    char value[PROPERTY_VALUE_MAX];  
    property_get("debug.sf.nobootanimation", value, "0");  
    int noBootAnimation = atoi(value);  
    LOGI_IF(noBootAnimation,  "boot animation disabled");  
    if (!noBootAnimation) {  

sp<ProcessState> proc(ProcessState::self());  
ProcessState::self()->startThreadPool();  

// create the boot animation object  
sp<BootAnimation> boot = new BootAnimation();  

IPCThreadState::self()->joinThreadPool();  

    }  
    return 0;  
}  
```
这个函数首先检查系统属性“debug.sf.nobootnimaition”的值是否不等于0。如果不等于的话，那么接下来就会启动一个Binder线程池，并且创建一个BootAnimation对象。这个BootAnimation对象就是用来显示第三个开机画面的。由于BootAnimation对象在显示第三个开机画面的过程中，需要与SurfaceFlinger服务通信，因此，应用程序bootanimation就需要启动一个Binder线程池。

BootAnimation类间接地继承了RefBase类，并且重写了RefBase类的成员函数onFirstRef，因此，当一个BootAnimation对象第一次被智能指针引用的时，这个BootAnimation对象的成员函数onFirstRef就会被调用。

BootAnimation类的成员函数onFirstRef实现在文件frameworks/base/cmds/bootanimation/BootAnimation.cpp中，如下所示：

```c
void BootAnimation::onFirstRef() {  
    status_t err = mSession->linkToComposerDeath(this);  
    LOGE_IF(err, "linkToComposerDeath failed (%s) ", strerror(-err));  
    if (err == NO_ERROR) {  
run("BootAnimation", PRIORITY_DISPLAY);  
    }  
}  

mSession是BootAnimation类的一个成员变量，它的类型为SurfaceComposerClient，是用来和SurfaceFlinger执行Binder进程间通信的，它是在BootAnimation类的构造函数中创建的，如下所示：

​```c
BootAnimation::BootAnimation() : Thread(false)  
{  
    mSession = new SurfaceComposerClient();  
}  
```
SurfaceComposerClient类内部有一个实现了ISurfaceComposerClient接口的Binder代理对象mClient，这个Binder代理对象引用了SurfaceFlinger服务，SurfaceComposerClient类就是通过它来和SurfaceFlinger服务通信的。

回到BootAnimation类的成员函数onFirstRef中，由于BootAnimation类引用了SurfaceFlinger服务，因此，当SurfaceFlinger服务意外死亡时，BootAnimation类就需要得到通知，这是通过调用成员变量mSession的成员函数linkToComposerDeath来注册SurfaceFlinger服务的死亡接收通知来实现的。

BootAnimation类继承了Thread类，因此，当BootAnimation类的成员函数onFirstRef调用了父类Thread的成员函数run之后，系统就会创建一个线程，这个线程在第一次运行之前，会调用BootAnimation类的成员函数readyToRun来执行一些初始化工作，后面再调用BootAnimation类的成员函数htreadLoop来显示第三个开机画面。

BootAnimation类的成员函数readyToRun的实现如下所示：

```c
status_t BootAnimation::readyToRun() {  
    mAssets.addDefaultAssets();  

    DisplayInfo dinfo;  
    status_t status = session()->getDisplayInfo(0, &dinfo);  
    if (status)  
return -1;  

    // create the native surface  
    sp<SurfaceControl> control = session()->createSurface(  
    getpid(), 0, dinfo.w, dinfo.h, PIXEL_FORMAT_RGB_565);  
    session()->openTransaction();  
    control->setLayer(0x40000000);  
    session()->closeTransaction();  

    sp<Surface> s = control->getSurface();  

    // initialize opengl and egl  
    const EGLint attribs[] = {  
    EGL_DEPTH_SIZE, 0,  
    EGL_NONE  
    };  
    EGLint w, h, dummy;  
    EGLint numConfigs;  
    EGLConfig config;  
    EGLSurface surface;  
    EGLContext context;  

    EGLDisplay display = eglGetDisplay(EGL_DEFAULT_DISPLAY);  

    eglInitialize(display, 0, 0);  
    EGLUtils::selectConfigForNativeWindow(display, attribs, s.get(), &config);  
    surface = eglCreateWindowSurface(display, config, s.get(), NULL);  
    context = eglCreateContext(display, config, NULL, NULL);  
    eglQuerySurface(display, surface, EGL_WIDTH, &w);  
    eglQuerySurface(display, surface, EGL_HEIGHT, &h);  

    if (eglMakeCurrent(display, surface, surface, context) == EGL_FALSE)  
return NO_INIT;  

    mDisplay = display;  
    mContext = context;  
    mSurface = surface;  
    mWidth = w;  
    mHeight = h;  
    mFlingerSurfaceControl = control;  
    mFlingerSurface = s;  

    mAndroidAnimation = true;  
    if ((access(USER_BOOTANIMATION_FILE, R_OK) == 0) &&  
    (mZip.open(USER_BOOTANIMATION_FILE) == NO_ERROR) ||  
    (access(SYSTEM_BOOTANIMATION_FILE, R_OK) == 0) &&  
    (mZip.open(SYSTEM_BOOTANIMATION_FILE) == NO_ERROR))  
mAndroidAnimation = false;  

    return NO_ERROR;  
}  
```
BootAnimation类的成员函数session用来返回BootAnimation类的成员变量mSession所描述的一个SurfaceComposerClient对象。通过调用SurfaceComposerClient对象mSession的成员函数createSurface可以获得一个SurfaceControl对象control。

SurfaceComposerClient类的成员函数createSurface首先调用内部的Binder代理对象mClient来请求SurfaceFlinger返回一个类型为SurfaceLayer的Binder代理对象，接着再使用这个Binder代理对象来创建一个SurfaceControl对象。创建出来的SurfaceControl对象的成员变量mSurface就指向了从SurfaceFlinger返回来的类型为SurfaceLayer的Binder代理对象。有了这个Binder代理对象之后，SurfaceControl对象就可以和SurfaceFlinger服务通信了。

调用SurfaceControl对象control的成员函数getSurface会返回一个Surface对象s。这个Surface对象s内部也有一个类型为SurfaceLayer的Binder代理对象mSurface，这个Binder代理对象与前面所创建的SurfaceControl对象control的内部的Binder代理对象mSurface引用的是同一个SurfaceLayer对象。这样，Surface对象s也可以通过其内部的Binder代理对象mSurface来和SurfaceFlinger服务通信。

Surface类继承了ANativeWindow类。ANativeWindow类是连接OpenGL和Android窗口系统的桥梁，即OpenGL需要通过ANativeWindow类来间接地操作Android窗口系统。这种桥梁关系是通过EGL库来建立的，所有以egl为前缀的函数名均为EGL库提供的接口。

为了能够在OpenGL和Android窗口系统之间的建立一个桥梁，我们需要一个EGLDisplay对象display，一个EGLConfig对象config，一个EGLSurface对象surface，以及一个EGLContext对象context，其中，EGLDisplay对象display用来描述一个EGL显示屏，EGLConfig对象config用来描述一个EGL帧缓冲区配置参数，EGLSurface对象surface用来描述一个EGL绘图表面，EGLContext对象context用来描述一个EGL绘图上下文（状态），它们是分别通过调用egl库函数eglGetDisplay、EGLUtils::selectConfigForNativeWindow、eglCreateWindowSurface和eglCreateContext来获得的。注意，EGLConfig对象config、EGLSurface对象surface和EGLContext对象context都是用来描述EGLDisplay对象display的。有了这些对象之后，就可以调用函数eglMakeCurrent来设置当前EGL库所使用的绘图表面以及绘图上下文。

还有另外一个地方需要注意的是，每一个EGLSurface对象surface有一个关联的ANativeWindow对象。这个ANativeWindow对象是通过函数eglCreateWindowSurface的第三个参数来指定的。在我们这个场景中，这个ANativeWindow对象正好对应于前面所创建的 Surface对象s。每当OpenGL需要绘图的时候，它就会找到前面所设置的绘图表面，即EGLSurface对象surface。有了EGLSurface对象surface之后，就可以找到与它关联的ANativeWindow对象，即Surface对象s。有了Surface对象s之后，就可以通过其内部的Binder代理对象mSurface来请求SurfaceFlinger服务返回帧缓冲区硬件设备的一个图形访问接口。这样，OpenGL最终就可以将要绘制的图形渲染到帧缓冲区硬件设备中去，即显示在实际屏幕上。屏幕的大小，即宽度和高度，可以通过函数eglQuerySurface来获得。

BootAnimation类的成员变量mAndroidAnimation是一个布尔变量。当它的值等于true的时候，那么就说明需要显示的第三个开机画面是Android系统默认的开机动画，否则的话，第三个开机画面就是由用户自定义的开机动画。

自定义的开机动画是由文件USER_BOOTANIMATION_FILE或者文件SYSTEM_BOOTANIMATION_FILE来描述的。只要其中的一个文件存在，那么第三个开机画面就会使用用户自定义的开机动画。USER_BOOTANIMATION_FILE和SYSTEM_BOOTANIMATION_FILE均是一个宏，它们的定义如下所示：

```c
#define USER_BOOTANIMATION_FILE "/data/local/bootanimation.zip"  
#define SYSTEM_BOOTANIMATION_FILE "/system/media/bootanimation.zip"  
```
这一步执行完成之后，用来显示第三个开机画面的线程的初始化工作就执行完成了，接下来，就会执行这个线程的主体函数，即BootAnimation类的成员函数threadLoop。

BootAnimation类的成员函数threadLoop的实现如下所示：

```c
bool BootAnimation::threadLoop()  
{  
    bool r;  
    if (mAndroidAnimation) {  
r = android();  
    } else {  
r = movie();  
    }  

    eglMakeCurrent(mDisplay, EGL_NO_SURFACE, EGL_NO_SURFACE, EGL_NO_CONTEXT);  
    eglDestroyContext(mDisplay, mContext);  
    eglDestroySurface(mDisplay, mSurface);  
    mFlingerSurface.clear();  
    mFlingerSurfaceControl.clear();  
    eglTerminate(mDisplay);  
    IPCThreadState::self()->stopProcess();  
    return r;  
}  
```
如果BootAnimation类的成员变量mAndroidAnimation的值等于true，那么接下来就会调用BootAnimation类的成员函数android来显示系统默认的开机动画，否则的话，就会调用BootAnimation类的成员函数movie来显示用户自定义的开机动画。显示完成之后，就会销毁前面所创建的EGLContext对象mContext、EGLSurface对象mSurface，以及EGLDisplay对象mDisplay等。

接下来，我们就分别分析BootAnimation类的成员函数android和movie的实现。

BootAnimation类的成员函数android的实现如下所示：

```c
bool BootAnimation::android()  
{  
    initTexture(&mAndroid[0], mAssets, "images/android-logo-mask.png");  
    initTexture(&mAndroid[1], mAssets, "images/android-logo-shine.png");  
    
    // clear screen  
    glShadeModel(GL_FLAT);  
    glDisable(GL_DITHER);  
    glDisable(GL_SCISSOR_TEST);  
    glClear(GL_COLOR_BUFFER_BIT);  
    eglSwapBuffers(mDisplay, mSurface);  
    
    glEnable(GL_TEXTURE_2D);  
    glTexEnvx(GL_TEXTURE_ENV, GL_TEXTURE_ENV_MODE, GL_REPLACE);  
    
    const GLint xc = (mWidth  - mAndroid[0].w) / 2;  
    const GLint yc = (mHeight - mAndroid[0].h) / 2;  
    const Rect updateRect(xc, yc, xc + mAndroid[0].w, yc + mAndroid[0].h);  
    
    // draw and update only what we need  
    mFlingerSurface->setSwapRectangle(updateRect);  
    
    glScissor(updateRect.left, mHeight - updateRect.bottom, updateRect.width(),  
    updateRect.height());  
    
    // Blend state  
    glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);  
    glTexEnvx(GL_TEXTURE_ENV, GL_TEXTURE_ENV_MODE, GL_REPLACE);  
    
    const nsecs_t startTime = systemTime();  
    do {  
nsecs_t now = systemTime();  
double time = now - startTime;  
float t = 4.0f * float(time / us2ns(16667)) / mAndroid[1].w;  
GLint offset = (1 - (t - floorf(t))) * mAndroid[1].w;  
GLint x = xc - offset;  

glDisable(GL_SCISSOR_TEST);  
glClear(GL_COLOR_BUFFER_BIT);  

glEnable(GL_SCISSOR_TEST);  
glDisable(GL_BLEND);  
glBindTexture(GL_TEXTURE_2D, mAndroid[1].name);  
glDrawTexiOES(x,                 yc, 0, mAndroid[1].w, mAndroid[1].h);  
glDrawTexiOES(x + mAndroid[1].w, yc, 0, mAndroid[1].w, mAndroid[1].h);  

glEnable(GL_BLEND);  
glBindTexture(GL_TEXTURE_2D, mAndroid[0].name);  
glDrawTexiOES(xc, yc, 0, mAndroid[0].w, mAndroid[0].h);  

EGLBoolean res = eglSwapBuffers(mDisplay, mSurface);  
if (res == EGL_FALSE) {  
    break;  
}  

// 12fps: don't animate too fast to preserve CPU  
const nsecs_t sleepTime = 83333 - ns2us(systemTime() - now);  
if (sleepTime > 0)  
    usleep(sleepTime);  
    } while (!exitPending());  
    
    glDeleteTextures(1, &mAndroid[0].name);  
    glDeleteTextures(1, &mAndroid[1].name);  
    return false;  
}  
```
Android系统默认的开机动画是由两张图片android-logo-mask.png和android-logo-shine.png中。这两张图片保存在frameworks/base/core/res/assets/images目录中，它们最终会被编译在framework-res模块（frameworks/base/core/res）中，即编译在framework-res.apk文件中。编译在framework-res模块中的资源文件可以通过AssetManager类来访问。

BootAnimation类的成员函数android首先调用另外一个成员函数initTexture来将根据图片android-logo-mask.png和android-logo-shine.png的内容来分别创建两个纹理对象，这两个纹理对象就分别保存在BootAnimation类的成员变量mAndroid所描述的一个数组中。通过混合渲染这两个纹理对象，我们就可以得到一个开机动画，这是通过中间的while循环语句来实现的。

图片android-logo-mask.png用作动画前景，它是一个镂空的“ANDROID”图像。图片android-logo-shine.png用作动画背景，它的中间包含有一个高亮的呈45度角的条纹。在每一次循环中，图片android-logo-shine.png被划分成左右两部分内容来显示。左右两个部分的图像宽度随着时间的推移而此消彼长，这样就可以使得图片android-logo-shine.png中间高亮的条纹好像在移动一样。另一方面，在每一次循环中，图片android-logo-shine.png都作为一个整体来渲染，而且它的位置是恒定不变的。由于它是一个镂空的“ANDROID”图像，因此，我们就可以通过它的镂空来看到它背后的图片android-logo-shine.png的条纹一闪一闪地划过。

这个while循环语句会一直被执行，直到应用程序/system/bin/bootanimation被结束为止，后面我们再分析。

BootAnimation类的成员函数movie的实现比较长，我们分段来阅读：

```c
bool BootAnimation::movie()  
{  
    ZipFileRO& zip(mZip);  

    size_t numEntries = zip.getNumEntries();  
    ZipEntryRO desc = zip.findEntryByName("desc.txt");  
    FileMap* descMap = zip.createEntryFileMap(desc);  
    LOGE_IF(!descMap, "descMap is null");  
    if (!descMap) {  
return false;  
    }  

    String8 desString((char const*)descMap->getDataPtr(),  
    descMap->getDataLength());  
    char const* s = desString.string();  

    Animation animation;  

    // Parse the description file  
    for (;;) {  
const char* endl = strstr(s, "\n");  
if (!endl) break;  
String8 line(s, endl - s);  
const char* l = line.string();  
int fps, width, height, count, pause;  
char path[256];  
if (sscanf(l, "%d %d %d", &width, &height, &fps) == 3) {  
    //LOGD("> w=%d, h=%d, fps=%d", fps, width, height);  
    animation.width = width;  
    animation.height = height;  
    animation.fps = fps;  
}  
if (sscanf(l, "p %d %d %s", &count, &pause, path) == 3) {  
    //LOGD("> count=%d, pause=%d, path=%s", count, pause, path);  
    Animation::Part part;  
    part.count = count;  
    part.pause = pause;  
    part.path = path;  
    animation.parts.add(part);  
}  
s = ++endl;  
    }  
```
从前面BootAnimation类的成员函数readyToRun的实现可以知道，如果目标设备上存在压缩文件/data/local/bootanimation.zip，那么BootAnimation类的成员变量mZip就会指向它，否则的话，就会指向目标设备上的压缩文件/system/media/bootanimation.zip。无论BootAnimation类的成员变量mZip指向的是哪一个压缩文件，这个压缩文件都必须包含有一个名称为“desc.txt”的文件，用来描述用户自定义的开机动画是如何显示的。

文件desc.txt的内容格式如下面的例子所示：

```
600 480 24  
p   1   0   part1  
p   0   10  part2  
```
第一行的三个数字分别表示开机动画在屏幕中的显示宽度、高度以及帧速（fps）。剩余的每一行都用来描述一个动画片断，这些行必须要以字符“p”来开头，后面紧跟着两个数字以及一个文件目录路径名称。第一个数字表示一个片断的循环显示次数，如果它的值等于0，那么就表示无限循环地显示该动画片断。第二个数字表示每一个片断在两次循环显示之间的时间间隔。这个时间间隔是以一个帧的时间为单位的。文件目录下面保存的是一系列png文件，这些png文件会被依次显示在屏幕中。

以上面这个desct.txt文件的内容为例，它描述了一个大小为600 x 480的开机动画，动画的显示速度为24帧每秒。这个开机动画包含有两个片断part1和part2。片断part1只显示一次，它对应的png图片保存在目录part1中。片断part2无限循环地显示，其中，每两次循环显示的时间间隔为10 x (1 / 24)秒，它对应的png图片保存在目录part2中。

上面的for循环语句分析完成desc.txt文件的内容后，就得到了开机动画的显示大小、速度以及片断信息。这些信息都保存在Animation对象animation中，其中，每一个动画片断都使用一个Animation::Part对象来描述，并且保存在Animation对象animation的成员变量parts所描述的一个片断列表中。

接下来，BootAnimation类的成员函数movie再断续将每一个片断所对应的png图片读取出来，如下所示：

```c
// read all the data structures  
const size_t pcount = animation.parts.size();  
for (size_t i=0 ; i<numEntries ; i++) {  
    char name[256];  
    ZipEntryRO entry = zip.findEntryByIndex(i);  
    if (zip.getEntryFileName(entry, name, 256) == 0) {  
const String8 entryName(name);  
const String8 path(entryName.getPathDir());  
const String8 leaf(entryName.getPathLeaf());  
if (leaf.size() > 0) {  
    for (int j=0 ; j<pcount ; j++) {  
        if (path == animation.parts[j].path) {  
            int method;  
            // supports only stored png files  
            if (zip.getEntryInfo(entry, &method, 0, 0, 0, 0, 0)) {  
                if (method == ZipFileRO::kCompressStored) {  
                    FileMap* map = zip.createEntryFileMap(entry);  
                    if (map) {  
                        Animation::Frame frame;  
                        frame.name = leaf;  
                        frame.map = map;  
                        Animation::Part& part(animation.parts.editItemAt(j));  
                        part.frames.add(frame);  
                    }  
                }  
            }  
        }  
    }  
}  
    }  
}  
```
每一个png图片都表示一个动画帧，使用一个Animation::Frame对象来描述，并且保存在对应的Animation::Part对象的成员变量frames所描述的一个帧列表中。

获得了开机动画的所有信息之后，接下来BootAnimation类的成员函数movie就准备开始显示开机动画了，如下所示：

```c
// clear screen  
glShadeModel(GL_FLAT);  
glDisable(GL_DITHER);  
glDisable(GL_SCISSOR_TEST);  
glDisable(GL_BLEND);  
glClear(GL_COLOR_BUFFER_BIT);  

eglSwapBuffers(mDisplay, mSurface);  

glBindTexture(GL_TEXTURE_2D, 0);  
glEnable(GL_TEXTURE_2D);  
glTexEnvx(GL_TEXTURE_ENV, GL_TEXTURE_ENV_MODE, GL_REPLACE);  
glTexParameterx(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT);  
glTexParameterx(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_REPEAT);  
glTexParameterx(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);  
glTexParameterx(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);  

const int xc = (mWidth - animation.width) / 2;  
const int yc = ((mHeight - animation.height) / 2);  
nsecs_t lastFrame = systemTime();  
nsecs_t frameDuration = s2ns(1) / animation.fps;  

Region clearReg(Rect(mWidth, mHeight));  
clearReg.subtractSelf(Rect(xc, yc, xc+animation.width, yc+animation.height));  
```
前面的一系列gl函数首先用来清理屏幕，接下来的一系列gl函数用来设置OpenGL的纹理显示方式。

变量xc和yc的值用来描述开机动画的显示位置，即需要在屏幕中间显示开机动画，另外一个变量frameDuration的值用来描述每一帧的显示时间，它是以纳秒为单位的。

Region对象clearReg用来描述屏幕中除了开机动画之外的其它区域，它是用整个屏幕区域减去开机动画所点据的区域来得到的。

准备好开机动画的显示参数之后，最后就可以执行显示开机动画的操作了，如下所示：

```c
    for (int i=0 ; i<pcount && !exitPending() ; i++) {  
const Animation::Part& part(animation.parts[i]);  
const size_t fcount = part.frames.size();  
glBindTexture(GL_TEXTURE_2D, 0);  

for (int r=0 ; !part.count || r<part.count ; r++) {  
    for (int j=0 ; j<fcount && !exitPending(); j++) {  
        const Animation::Frame& frame(part.frames[j]);  

        if (r > 0) {  
            glBindTexture(GL_TEXTURE_2D, frame.tid);  
        } else {  
            if (part.count != 1) {  
                glGenTextures(1, &frame.tid);  
                glBindTexture(GL_TEXTURE_2D, frame.tid);  
                glTexParameterx(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);  
                glTexParameterx(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);  
            }  
            initTexture(  
                    frame.map->getDataPtr(),  
                    frame.map->getDataLength());  
        }  

        if (!clearReg.isEmpty()) {  
            Region::const_iterator head(clearReg.begin());  
            Region::const_iterator tail(clearReg.end());  
            glEnable(GL_SCISSOR_TEST);  
            while (head != tail) {  
                const Rect& r(*head++);  
                glScissor(r.left, mHeight - r.bottom,  
                        r.width(), r.height());  
                glClear(GL_COLOR_BUFFER_BIT);  
            }  
            glDisable(GL_SCISSOR_TEST);  
        }  
        glDrawTexiOES(xc, yc, 0, animation.width, animation.height);  
        eglSwapBuffers(mDisplay, mSurface);  

        nsecs_t now = systemTime();  
        nsecs_t delay = frameDuration - (now - lastFrame);  
        lastFrame = now;  
        long wait = ns2us(frameDuration);  
        if (wait > 0)  
            usleep(wait);  
    }  
    usleep(part.pause * ns2us(frameDuration));  
}  

// free the textures for this part  
if (part.count != 1) {  
    for (int j=0 ; j<fcount ; j++) {  
        const Animation::Frame& frame(part.frames[j]);  
        glDeleteTextures(1, &frame.tid);  
    }  
}  
    }  

    return false;  
```

第一层for循环用来显示每一个动画片断，第二层的for循环用来循环显示每一个动画片断，第三层的for循环用来显示每一个动画片断所对应的png图片。这些png图片以纹理的方式来显示在屏幕中。

注意，如果一个动画片断的循环显示次数不等于1，那么就说明这个动画片断中的png图片需要重复地显示在屏幕中。由于每一个png图片都需要转换为一个纹理对象之后才能显示在屏幕中，因此，为了避免重复地为同一个png图片创建纹理对象，第三层的for循环在第一次显示一个png图片的时候，会调用函数glGenTextures来为这个png图片创建一个纹理对象，并且将这个纹理对象的名称保存在对应的Animation::Frame对象的成员变量tid中，这样，下次再显示相同的图片时，就可以使用前面已经创建好了的纹理对象，即调用函数glBindTexture来指定当前要操作的纹理对象。

如果Region对象clearReg所包含的区域不为空，那么在调用函数glDrawTexiOES和eglSwapBuffers来显示每一个png图片之前，首先要将它所包含的区域裁剪掉，避免开机动画可以显示在指定的位置以及大小中。

每当显示完成一个png图片之后，都要将变量frameDuration的值从纳秒转换为毫秒。如果转换后的值大小于，那么就需要调用函数usleep函数来让线程睡眠一下，以保证每一个png图片，即每一帧动画都按照预先指定好的速度来显示。注意，函数usleep指定的睡眠时间只能精确到毫秒，因此，如果预先指定的帧显示时间小于1毫秒，那么BootAnimation类的成员函数movie是无法精确地控制地每一帧的显示时间的。

还有另外一个地方需要注意的是，每当循环显示完成一个片断时，需要调用usleep函数来使得线程睡眠part.pause * ns2us(frameDuration)毫秒，以便可以按照预先设定的节奏来显示开机动画。

最后一个if语句判断一个动画片断是否是循环显示的，即循环次数不等于1。如果是的话，那么就说明前面为它所对应的每一个png图片都创建过一个纹理对象。现在既然这个片断的显示过程已经结束了，因此，就需要释放前面为它所创建的纹理对象。

至此，第三个开机画面的显示过程就分析完成了。

接下来，我们再继续分析第三个开机画面是如何停止显示的。

从前面[Android系统默认Home应用程序（Launcher）的启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6767736)一文可以知道，当System进程将系统中的关键服务启动起来之后，就会将应用程序启动器（Launcher）启动起来。从[Android应用程序启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6689748)一文又可以知道，Android应用程序的启动过程实际上就是它的根Activity组件的启动过程。对于应用程序Launcher来说，它的根Activity组件即为Launcher组件。

一个Activity组件在启动起来之后，就会被记录起来，等到它所运行在的主线程空闲的时候，这个主线程就会向ActivityManagerService发送一个Activity组件空闲的通知。由于应用程序Launcher是系统中第一个被启动的应用程序，即它的根Activity组件是系统中第一个被启动的Activity组件，因此，当ActivityManagerService接收到它的空闲通知的时候，就可以知道系统是刚刚启动起来的。在这种情况下，ActivityManagerService就会停止显示开机动画，以便可以在屏幕中显示应用程序Lancher的界面。

从前面[Android应用程序消息处理机制（Looper、Handler）分析](http://blog.csdn.net/luoshengyang/article/details/6817933)一文可以知道，如果一个线程想要在空闲的时候处理一些事务，那么就必须要向这个线程的消息队列注册一个空闲消息处理器。自定义的空闲消息处理器灯必须要从MessageQueue.IdleHandler类继承下来，并且重写成员函数queueIdle。当一个线程空闲的时候，即消息队列中没有新的消息需要处理的时候，那些注册了的空闲消息处理器的成员函数queueIdle就会被调用。

应用程序的主线程是通过ActivityThread类来描述的，它实现在文件frameworks/base/core/[Java](http://lib.csdn.net/base/java)/android/app/ActivityThread.java中。每当有一个新的Activity组件启动起来的时候，ActivityThread类都会向它所描述的应用程序主线程的消息队列注册一个类型为Idler的空闲消息处理器。这样一个应用程序的主线程就可以在空闲的时候，向ActivityManagerService发送一个Activity组件空闲通知，相当于是通知ActivityManagerService，一个新的Activity组件已经准备就绪了。

Idler类定义在frameworks/base/core/java/android/app/ActivityThread.java中， 它的成员函数queueIdle的实现如下所示： 

```java
public final class ActivityThread {  
    ......  

    private final class Idler implements MessageQueue.IdleHandler {  
public final boolean queueIdle() {  
    ActivityClientRecord a = mNewActivities;  
    if (a != null) {  
        mNewActivities = null;  
        IActivityManager am = ActivityManagerNative.getDefault();  
        ActivityClientRecord prev;  
        do {  
            ......  
            if (a.activity != null && !a.activity.mFinished) {  
                try {  
                    am.activityIdle(a.token, a.createdConfig);  
                    a.createdConfig = null;  
                } catch (RemoteException ex) {  
                }  
            }  
            prev = a;  
            a = a.nextIdle;  
            prev.nextIdle = null;  
        } while (a != null);  
    }  
    ensureJitEnabled();  
    return false;  
}  
    }  

    ......  
}  
```
ActivityThread类有一个类型为ActivityClientRecord的成员变量mNewActivities，用来描述所有在当前应用程序主线程中新启动起来的Activity组件。这些新启动起来的Activity组件通过ActivityClientRecord类的成员变量nextIdle连接在一起。一旦当前应用程序主线程向ActivityManagerService发送了这些新启动的Activity组件的空闲通知之后，这些新启动起来的Activity组件就不会再被保存在ActivityThread类的成员变量mNewActivities中了，即每一个新启动的Activity组件只有一次机会向ActivityManagerService发送一个空闲通知。

向ActivityManagerService发送一个Activity组件空闲通知是通过调用ActivityManagerService代理对象的成员函数activityIdle来实现的，而ActivityManagerService代理对象可以通过调用ActivityManagerNative类的静态成员函数getDefault来获得。

ActivityManagerService代理对象的类型为ActivityManagerProxy，它的成员函数activityIdle实现在文件frameworks/base/core/java/android/app/ActivityManagerNative.java中，如下所示：

```java
class ActivityManagerProxy implements IActivityManager  
{  
    ......  

    public void activityIdle(IBinder token, Configuration config) throws RemoteException  
    {  
Parcel data = Parcel.obtain();  
Parcel reply = Parcel.obtain();  
data.writeInterfaceToken(IActivityManager.descriptor);  
data.writeStrongBinder(token);  
if (config != null) {  
    data.writeInt(1);  
    config.writeToParcel(data, 0);  
} else {  
    data.writeInt(0);  
}  
mRemote.transact(ACTIVITY_IDLE_TRANSACTION, data, reply, IBinder.FLAG_ONEWAY);  
reply.readException();  
data.recycle();  
reply.recycle();  
    }  

    ......  
}  
```
ActivityManagerProxy类的成员函数activityIdle实际上是向ActivityManagerService发送一个类型为ACTIVITY_IDLE_TRANSACTION的Binder进程间通信请求，其中，参数token用来描述与这个进程间通信请求所关联的一个Activity组件，在我们这个场景中，这个Activity组件即为应用程序Launcher的根Activity组件Launcher。

类型为ACTIVITY_IDLE_TRANSACTION的Binder进程间通信请求是由ActivityManagerService类的成员函数activityIdle来处理的，如下所示：

```java
public final class ActivityManagerService extends ActivityManagerNative  
implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {  
    ......  

    public final void activityIdle(IBinder token, Configuration config) {  
final long origId = Binder.clearCallingIdentity();  
mMainStack.activityIdleInternal(token, false, config);  
Binder.restoreCallingIdentity(origId);  
    }  

    ......  
}  
```
ActivityManagerService类有一个类型为ActivityStack的成员变量mMainStack，它用来描述系统的Activity组件堆栈，它的成员函数activityIdleInternal的实现如下所示：

```java
public class ActivityStack {  
    ......  

    final void activityIdleInternal(IBinder token, boolean fromTimeout,  
    Configuration config) {  
......  

boolean enableScreen = false;  

synchronized (mService) {  
    ......  

    // Get the activity record.  
    int index = indexOfTokenLocked(token);  
    if (index >= 0) {  
        ActivityRecord r = (ActivityRecord)mHistory.get(index);                  
        ......  

        if (mMainStack) {  
            if (!mService.mBooted && !fromTimeout) {  
                mService.mBooted = true;  
                enableScreen = true;  
            }  
        }  
    }  

    ......  
}  

......  

if (enableScreen) {  
    mService.enableScreenAfterBoot();  
}  
    }  

    ......  
}          
```
参数token用来描述刚刚启动起来的Launcher组件，通过它来调用函数indexOfTokenLocked就可以得到Launcher组件在系统Activity组件堆栈中的位置index。得到了Launcher组件在系统Activity组件堆栈中的位置index之后，就可以在ActivityStack类的成员变量mHistory中得到一个ActivityRecord对象r。这个ActivityRecord对象r同样是用来描述Launcher组件的。

ActivityStack类的成员变量mMainStack是一个布尔变量，当它的值等于true的时候，就说明当前正在处理的ActivityStack对象是用来描述系统的Activity组件堆栈的。 ActivityStack类的另外一个成员变量mService指向了系统中的ActivityManagerService服务。ActivityManagerService服务有一个类型为布尔值的成员变量mBooted，它的初始值为false，表示系统尚未启动完成。

从前面的调用过程可以知道，参数fromTimeout的值等于false。在这种情况下，如果ActivityManagerService服务的成员变量mBooted也等于false，那么就说明应用程序已经启动起来了，即说明系统已经启动完成了。这时候ActivityManagerService服务的成员变量mBooted以及变量enableScreen的值就会被设置为true。

当变量enableScreen的值等于true的时候，ActivityStack类就会调用ActivityManagerService服务的成员函数enableScreenAfterBoot停止显示开机动画，以便可以将屏幕让出来显示应用程序Launcher的界面。

ActivityManagerService类的成员函数enableScreenAfterBoot的实现如下所示：

```java
public final class ActivityManagerService extends ActivityManagerNative  
implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {  
    ......  

    void enableScreenAfterBoot() {  
EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_ENABLE_SCREEN,  
        SystemClock.uptimeMillis());  
mWindowManager.enableScreenAfterBoot();  
    }  

    ......  
}  
```
ActivityManagerService类的成员变量mWindowManager指向了系统中的Window管理服务WindowManagerService，ActivityManagerService服务通过调用它的成员函数enableScreenAfterBoot来停止显示开机动画。

WindowManagerService类的成员函数enableScreenAfterBoot的实现如下所示：

```java
public class WindowManagerService extends IWindowManager.Stub  
implements Watchdog.Monitor {  
    ......  

    public void enableScreenAfterBoot() {  
synchronized(mWindowMap) {  
    if (mSystemBooted) {  
        return;  
    }  
    mSystemBooted = true;  
}  

performEnableScreen();  
    }  

    ......  
}  
```
WindowManagerService类的成员变量mSystemBooted用来记录系统是否已经启动完成的。如果已经启动完成的话，那么这个成员变量的值就会等于true，这时候WindowManagerService类的成员函数enableScreenAfterBoot什么也不做就返回了，否则的话，WindowManagerService类的成员函数enableScreenAfterBoot首先将这个成员变量的值设置为true，接着再调用另外一个成员函数performEnableScreen来执行停止显示开机动画的操作。

WindowManagerService类的成员函数performEnableScreen的实现如下所示：

```java
public class WindowManagerService extends IWindowManager.Stub  
implements Watchdog.Monitor {  
    ......  

    public void performEnableScreen() {  
synchronized(mWindowMap) {  
    if (mDisplayEnabled) {  
        return;  
    }  
    if (!mSystemBooted) {  
        return;  
    }  

    ......  

    mDisplayEnabled = true;  
    ......  

    try {  
        IBinder surfaceFlinger = ServiceManager.getService("SurfaceFlinger");  
        if (surfaceFlinger != null) {  
            //Slog.i(TAG, "******* TELLING SURFACE FLINGER WE ARE BOOTED!");  
            Parcel data = Parcel.obtain();  
            data.writeInterfaceToken("android.ui.ISurfaceComposer");  
            surfaceFlinger.transact(IBinder.FIRST_CALL_TRANSACTION,  
                                    data, null, 0);  
            data.recycle();  
        }  
    } catch (RemoteException ex) {  
        Slog.e(TAG, "Boot completed: SurfaceFlinger is dead!");  
    }  
}  

......  
    }  

    ......  
}  
```
WindowManagerService类的另外一个成员变量mDisplayEnabled用来描述WindowManagerService是否已经初始化过系统的屏幕了，只有当它的值等于false，并且系统已经完成启动，即WindowManagerService类的成员变量mSystemBooted等于true的情况下，WindowManagerService类的成员函数performEnableScreen才通知SurfaceFlinger服务停止显示开机动画。

注意，WindowManagerService类的成员函数performEnableScreen是通过一个类型为IBinder.FIRST_CALL_TRANSACTION的进程间通信请求来通知SurfaceFlinger服务停止显示开机动画的。

在SurfaceFlinger服务，类型为IBinder.FIRST_CALL_TRANSACTION的进程间通信请求被定义为停止显示开机动画的请求，如下所示：

```c
class BnSurfaceComposer : public BnInterface<ISurfaceComposer>  
{  
public:  
    enum {  
// Note: BOOT_FINISHED must remain this value, it is called from  
// Java by ActivityManagerService.  
BOOT_FINISHED = IBinder::FIRST_CALL_TRANSACTION,  
......  
    };  

    virtual status_t    onTransact( uint32_t code,  
                            const Parcel& data,  
                            Parcel* reply,  
                            uint32_t flags = 0);  
};  
```
BnSurfaceComposer类定义在文件frameworks/base/include/surfaceflinger/ISurfaceComposer.h中，它是SurfaceFlinger服务所要继承的Binder本地对象类，其中。当SurfaceFlinger服务接收到类型为IBinder::FIRST_CALL_TRANSACTION，即类型为BOOT_FINISHED的进程间通信请求时，它就会将该请求交给它的成员函数bootFinished来处理。

SurfaceFlinger服务的成员函数bootFinished实现在文件frameworks/base/services/surfaceflinger/SurfaceFlinger.cpp中，如下所示：

```c
void SurfaceFlinger::bootFinished()  
{  
    const nsecs_t now = systemTime();  
    const nsecs_t duration = now - mBootTime;  
    LOGI("Boot is finished (%ld ms)", long(ns2ms(duration)) );  
    mBootFinished = true;  
    property_set("ctl.stop", "bootanim");  
}  
```
这个函数主要就是将系统属性“ctl.stop”的值设置为“bootanim”。前面提到，每当有一个系统属性发生变化时，init进程就会被唤醒，并且调用运行在它里面的函数handle_property_set_fd来处理这个系统属性变化事件。在我们这个场景中，由于被改变的系统属性的名称是以"ctl."开头的，即被改变的系统属性是一个控制类型的属性，因此，接下来函数handle_property_set_fd又会调用另外一个函数handle_control_message来处理该系统属性变化事件。

函数handle_control_message实现在文件system/core/init/init.c中，如下所示：

```c
void handle_control_message(const char *msg, const char *arg)  
{  
    if (!strcmp(msg,"start")) {  
msg_start(arg);  
    } else if (!strcmp(msg,"stop")) {  
msg_stop(arg);  
    } else {  
ERROR("unknown control msg '%s'\n", msg);  
    }  
}  
```
从前面的调用过程可以知道，参数msg和arg的值分别等于"stop"和“bootanim”，这表示要停止执行名称为“bootanim”的服务，这是通过调用函数msg_stop来实现的。   

函数msg_stop也是实现在文件system/core/init/init.c中，如下所示：

```c
static void msg_stop(const char *name)  
{  
    struct service *svc = service_find_by_name(name);  

    if (svc) {  
service_stop(svc);  
    } else {  
ERROR("no such service '%s'\n", name);  
    }  
}  
```
这个函数首先调用函数service_find_by_name来找到名称等于name，即“bootanim”的服务，然后再调用函数service_stop来停止这个服务。

前面提到，名称为“bootanim”的服务对应的应用程序即为/system/bin/bootanimation。因此，停止名称为“bootanim”的服务即为停止执行应用程序/system/bin/bootanimation，而当应用程序/system/bin/bootanimation停止执行的时候，开机动画就会停止显示了。

至此，Android系统的三个开机画面的显示过程就分析完成了。通过这个三个开机画面的显示过程分析，我们学习到：

1. 在内核层，系统屏幕是使用一个称为帧缓冲区的硬件设备来描述的，而用户空间的应用程序可以通过设备文件/dev/fb0或者/dev/graphics/fb0来操作这个硬件设备。实际上，帧缓冲区本身并不是一个真正的硬件，它只不过是对显卡的一个抽象表示，不过，我们通过访帧缓冲区就可以间接地操作显卡内存以及显卡中的其它寄存器。

2. OpenGL是通过EGL接口来渲染屏幕，而EGL接口是通过ANativeWindow类来间接地渲染屏幕的。我们可以将ANativeWindow类理解成一个Android系统的本地窗口类，即相当于是Windows系统中的窗口句柄概念，它最终是通过文件/dev/fb0或者/dev/graphics/fb0来渲染屏幕的。

3. init进程在启动的过程中，会将另外一个ueventd进程也启动起来。ueventd进程对应的可执行文件与init进程对应的可执行文件均为/init，不过ueventd进程主要负责处理内核发出的uevent事件，即负责管理系统中的设备文件。

4. 每当我们设置一个系统属性的时候，init进程都会接收到一个系统属性变化事件。当发生变化的系统属性的名称等于“ctl.start”或者“ctl.stop”，那么实际上是向init进程发出一个启动或者停止服务的命令。

前面第1点和第2点的知识是与Android系统的UI实现相关的，而后面第3点和第4点是两个额外获得的知识点。

本文的目的并不是单纯为了介绍Android系统的开机画面，而是希望能够以Android系统的开机画面来作为切入点来分析Android系统的UI实现。在后面的文章中，我们就会根据本文所涉及到的知识点，来展开分析Android系统的UI实现，敬请关注。
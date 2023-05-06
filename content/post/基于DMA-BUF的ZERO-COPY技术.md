---
layout:     post

title:      "基于DMA-BUF的ZERO-COPY技术"
subtitle:   ""
excerpt: ""
author:     "button"
date:       2023-04-28
published: true 
tags:
    - 工程实战 

categories: [ Tech ]
URL: "/2023/04/28/"
---


> 现在做一个车机项目，是基于Linux系统上面的wayland应用，目前有一个瓶颈在于从client到server这边图形数据的传送需要通过glReadPixels()先从gpu回读到cpu，然后通过memcpy到server，其中gpu回读到cpu的耗时很大，大概要几十ms，如果是多屏，那就更多了，而且会堵塞rt线程，目前就是要解决这个瓶颈!

## 一 前言

现有方案如下：
![现有方案](/img/wayland%E5%A4%9A%E5%B1%8F%E6%98%BE%E7%A4%BA%E7%8E%B0%E6%9C%89%E6%96%B9%E6%A1%88.png)

改进方案1如下：
![改进方案1](/img/wayland%E5%A4%9A%E5%B1%8F%E6%98%BE%E7%A4%BA%E6%94%B9%E8%BF%9B%E6%96%B9%E6%A1%881.png)

改进方案2如下：
![改进方案2](/img/wayland%E5%A4%9A%E5%B1%8F%E6%98%BE%E7%A4%BA%E6%94%B9%E8%BF%9B%E6%96%B9%E6%A1%882.png)

## 二 问题
单刀直入，这里需要解决两个问题：

* 1 Wayland Client和Wayland Server端的内存共享问题；

* 2 GPU显存和Client端的内存打通问题；

## 三 解决方案 
### 问题1解决
那么先来解决问题1，let‘s go！
```
int width = w->cfg.width;

    int height = w->cfg.height;

    auto renderer = new struct cwl_dmabuf_renderer;

    c->renderer_data = renderer;

    auto extension = (cwl_linux_dmabuf_extension*) cwl_find_extension(c, "zwp_linux_dmabuf_v1");

    cwl_assert(extension != nullptr);

    renderer->cwl = c;

    renderer->extension = extension;

    extension->set_callbacks(&cwl_dmabuf_render_ext_callbacks, renderer);

    //struct cwl_dmabuf_renderer *renderer = (struct cwl_dmabuf_renderer *) c->renderer_data;

    cwl_assert(renderer != nullptr);

    //auto extension = renderer->extension;

    auto gbm_window = new cwl_gbm_window;

    gbm_window->window = w;

    w->renderer_data = gbm_window;

    uint32_t flags = ZWP_LINUX_BUFFER_PARAMS_V1_FLAGS_Y_INVERT; // TODO

    int format = DRM_FORMAT_XRGB8888;  // TODO

    #define DEFAULT_DEVICE_NAME "/dev/dri/renderD128"

    const char *device_name = DEFAULT_DEVICE_NAME;

    if (getenv("CWL_MAIN_DEVICE") != nullptr) {

        device_name = getenv("CWL_MAIN_DEVICE");

    }

    cwl_gbm_init(&renderer->gbm, device_name);

struct cwl_gbm_buffer *gbm_buffer = &gbm_window->gbm_buffers[i];

    cwl_gbm_init_buffer(&renderer->gbm, gbm_buffer, extension, width, height, format);

    auto params = zwp_linux_dmabuf_v1_create_params(extension->dmabuf);

    zwp_linux_buffer_params_v1_add_listener(params, &cwl_linux_buffer_params_v1_listener, (void *) gbm_buffer);

    for (int j = 0; j < gbm_buffer->plane_count; j++) {

            zwp_linux_buffer_params_v1_add(params,

            gbm_buffer->planes[j].dmabuf_fd,

                j /* plane_idx */,

                gbm_buffer->planes[j].offset,

                gbm_buffer->planes[j].stride,

                gbm_buffer->modifier >> 32,

                gbm_buffer->modifier & 0xffffffff);

        }

        wl_buffer *buffer = zwp_linux_buffer_params_v1_create_immed(params,

                                width, height, format, flags);

        cwl_assert(buffer);

        uint8_t *pixel_data = gbm_buffer->pixel_data;//no need to use

        uint32_t buffer_id = wl_proxy_get_id((wl_proxy *) buffer);

cwl_dmabuf_buffer *buf = (cwl_dmabuf_buffer *) data;

    cwl_assert(buf != nullptr && buf->buffer.in_use);

wl_surface_attach(w->wl.surface, buf->buffer.window->current->buffer.wl_buffer, 0, 0);

    //wl_surface_attach(w->wl.surface, buf->buffer.wl_buffer, 0, 0);

    wl_surface_damage_buffer(w->wl.surface, 0, 0, w->cfg.width, w->cfg.height);

    w->wl.frame_callback = wl_surface_frame(w->wl.surface);

    wl_callback_add_listener(w->wl.frame_callback, &cwl_dmabuf_frame_listener, w);

    wl_surface_commit(w->wl.surface);
```

以上代码可以看到，通过GBM，我们创建了一块基于DMA-BUF的wl-buffer，接着就是提交给wayland-server add，GBM这边的init代码如下：
```

if (!gbm_buffer->gbm_bo) {

        gbm_buffer->gbm_bo = gbm_bo_create(gbm->device,

            width, height, format, GBM_BO_USE_RENDERING|GBM_BO_USE_LINEAR);

if (!gbm_buffer->gbm_bo) {

            return false;

        }

        gbm_buffer->modifier = gbm_bo_get_modifier(gbm_buffer->gbm_bo);

        //fprintf(stderr, "LZ: cwl_gbm_init_buffer/gbm_bo_create, pGbmBo = %p \n", gbm_buffer->gbm_bo);

    }

    gbm_buffer->plane_count = gbm_bo_get_plane_count(gbm_buffer->gbm_bo);

    for (int i = 0; i < gbm_buffer->plane_count; i++) {

        union gbm_bo_handle handle;

        handle = gbm_bo_get_handle_for_plane(gbm_buffer->gbm_bo, i);

        if (handle.s32 == -1) {

            return false;

        }

        int ret = drmPrimeHandleToFD(gbm->drm_fd, handle.u32, 0, &gbm_buffer->planes[i].dmabuf_fd);

        if (ret < 0 || gbm_buffer->planes[i].dmabuf_fd < 0) {

            return false;

        }

        gbm_buffer->planes[i].stride = gbm_bo_get_stride_for_plane(gbm_buffer->gbm_bo, i);

        //gbm_buffer->planes[i].stride = width * 4;//gbm_bo_get_stride_for_plane(gbm_buffer->gbm_bo, i);

        gbm_buffer->planes[i].offset = gbm_bo_get_offset(gbm_buffer->gbm_bo, i);

    }
```
这样问题1就解决了，实现了基于DMA-BUF的client端到server端的通讯,server端会直接拿到DMA-BUF的FD，可以GPU直接用来显示；

具体相关Linux协议详见[Linux DMA-BUF protocol | Wayland Explorer](https://wayland.app/protocols/linux-dmabuf-unstable-v1)

### 问题2解决

接着解决问题2，问题2会稍微麻烦点，我尝试了两种方式。

#### 方法1——基于Linux协议的MESA_image_dma_buf_export扩展
```
share->image = egl->eglCreateImageKHR(egl_display,

                            egl_context,

                            EGL_GL_TEXTURE_2D,

                            (EGLClientBuffer)(uint64_t) share->tex,

                            attribs);

    egl->eglExportDMABUFImageQueryMESA(egl_display,

                                    share->image,

                                    &share->texinfo.fourcc,

                                    NULL,

                                    NULL);

    egl->eglExportDMABUFImageMESA(egl_display,

                                share->image,

                                &share->dmabuf_fd,

                                &share->texinfo.stride,

                                &share->texinfo.offset);
```
可惜的是，这个extension依赖于高通的芯片驱动支持，但是短期内无法上线，需要找其他出路了；

那么方式2来了

#### 方放2——基于Linux协议的MESA_image_dma_buf_import扩展
```
attribs[index++] = EGL_WIDTH;

    attribs[index++] = out->w;

    attribs[index++] = EGL_HEIGHT;

    attribs[index++] = out->h;

    attribs[index++] = EGL_LINUX_DRM_FOURCC_EXT;

    attribs[index++] = DRM_FORMAT_XRGB8888;

    #define addPlane(plane_idx)                                                        \

            do { \

            attribs[index++] = EGL_DMA_BUF_PLANE ## plane_idx ## _FD_EXT;     \

            attribs[index++] = pGbmBuf->planes[plane_idx].dmabuf_fd;         \

            attribs[index++] = EGL_DMA_BUF_PLANE ## plane_idx ## _OFFSET_EXT;     \

            attribs[index++] = pGbmBuf->planes[plane_idx].offset;         \

            attribs[index++] = EGL_DMA_BUF_PLANE ## plane_idx ## _PITCH_EXT;     \

            attribs[index++] = pGbmBuf->planes[plane_idx].stride;         \

            attribs[index++] = EGL_DMA_BUF_PLANE ## plane_idx ## _MODIFIER_LO_EXT;     \

            attribs[index++] = (uint32_t)(pGbmBuf->modifier & 0xffffffff);         \

            attribs[index++] = EGL_DMA_BUF_PLANE ## plane_idx ## _MODIFIER_HI_EXT;     \

            attribs[index++] = (uint32_t)(pGbmBuf->modifier >> 32);             \

        } while (0)

    if (pGbmBuf->plane_count > 0) addPlane(0);

    if (pGbmBuf->plane_count > 1) addPlane(1);

    if (pGbmBuf->plane_count > 2) addPlane(2);

    if (pGbmBuf->plane_count > 3) addPlane(3);

    #undef addPlane

    attribs[index] = EGL_NONE;

    out->eglDisplay = egl->eglGetCurrentDisplay();

    out->eglContext = egl->eglGetCurrentContext();

    out->eglImage = egl->eglCreateImage(out->eglDisplay,

        out->eglContext,

        EGL_LINUX_DMA_BUF_EXT,

        0,

        attribs);

gl->glBindTexture(GL_TEXTURE_2D, texinfo);

gl->glEGLImageTargetTexture2DOES(GL_TEXTURE_2D, out->eglImage);
```

**这里需要注意的是egl-context，还有线程间的切换，必须在rt线程-**

**下面两行是关键，将DMA-BUF和GPU的显存区域绑定起来，这样GPU直接渲染到DMA-BUF，然后client端将DMA-BUF的FD传递给server端，全程都是GPU操作，没有任何多余的copy操作，俗称zero-copy。**

调试成功后发现也没啥，但是调试过程涉及到图形texture，framebuffer，egl context，gpu driver，DMA-BUF各种技术，中间调试中出现的最大的问题是
```
gbm_buffer->planes[i].stride = gbm_bo_get_stride_for_plane(gbm_buffer->gbm_bo, i);
```
这个参数被之前的同事解一个花屏的问题改成width*4，导致中间怎么调试都写入不了任何数据，包括最普通的全屏红色都写入不了，几乎确认方案不可行，差点放弃。由此得到的教训是系统api千万别用野路子去替代，就算当前没有问题，后面也要还债，而且一旦出问题后面很难查出来！


https://www.jianshu.com/p/d17396696e9b

import { Injectable, ApplicationRef, ComponentFactoryResolver,
    ComponentRef, EmbeddedViewRef } from '@angular/core';
import { YupRef, ComponentType } from './popup.ref';

@Injectable()
export class DialogService {
    private loadRef: YupRef<LoadComponent>;
    constructor(
        private appRef: ApplicationRef,
        private compFactRes: ComponentFactoryResolver
    ) {}
    // 创建一个组件，组件通过泛型传入以做到通用
    public open<T>(component: ComponentType<T>, config: any) {
        // 创建组件工厂
        const factory = this.compFactRes.resolveComponentFactory(component);
        // 创建一个新的弹窗引用
        const dialogRef = new YupRef(factory, config);
        // 将创建好的组件引用(由弹窗引用创建并返回)append到body标签下
        window.document.body.appendChild(this.getComponentRootNode(dialogRef.componentRef()));
        // 加入angular脏检查
        this.appRef.attachView(dialogRef.componentRef().hostView);
        // 将创建的弹窗引用返回给外界
        return dialogRef;
    }
    // 参考自Material2，将ComponentRef类型的组件引用转换为DOM节点
    private getComponentRootNode(componentRef: ComponentRef<any>): HTMLElement {
        return (componentRef.hostView as EmbeddedViewRef<any>).rootNodes[0] as HTMLElement;
    }
}

// 参考自Material2 用于作为传入组件的类型
export interface ComponentType<T> {
    new (...args: any[]): T;
}











import { ComponentRef, InjectionToken, ReflectiveInjector, ComponentFactory } from '@angular/core';
import { Observable } from 'rxjs/Observable';
import { Subject } from 'rxjs/Subject';
// 用于注入自定义数据到创建的组件中
export const YUP_DATA = new InjectionToken<any>('YUPPopupData');

export class YupRef<T> {
    // 弹窗关闭的订阅
    private afterClose$: Subject<any>;
    // 弹窗引用变量
    private dialogRef: ComponentRef<T>;
    constructor(
        private factory: ComponentFactory<T>,
        private config: any // 传入的自定义数据
    ) {
        this.afterClose$ = new Subject<any>();
        this.dialogRef = this.factory.create(
            ReflectiveInjector.resolveAndCreate([
                {provide: YUP_DATA, useValue: config}, // 注入自定义数据
                {provide: YupRef, useValue: this} // 注入自身，这样就可以在创建的组件里控制组件的关闭等
            ])
        );
    }
    // 提供给外界的对窗口关闭的订阅
    public afterClose(): Observable<any> {
        return this.afterClose$.asObservable();
    }
    // 关闭方法，将销毁组件
    public close(data?: any) {
        this.afterClose$.next(data);
        this.afterClose$.complete();
        this.dialogRef.destroy();
    }
    // 提供给弹窗服务以帮助添加到DOM中
    public componentRef() {
        return this.dialogRef;
    }
}










import { Component, Injector } from '@angular/core';
import { YupRef, YUP_DATA } from '../popup.ref';
import { mask, dialog } from '../animations';

@Component({
    template: `
    <div class="yup-mask" [@mask]="disp" (click)="!data?.mask && close(false)"></div>
    <div class="yup-body" [@dialog]="disp">
        <div class="yup-body-head">{{data?.title || '消息'}}</div>
        <div class="yup-body-content">{{data?.msg || ' '}}</div>
        <div class="yup-body-btns">
            <div class="btn default" (click)="close(false)">{{data?.no || '取消'}}</div>
            <div class="btn primary" (click)="close(true)">{{data?.ok || '确认'}}</div>
        </div>
    </div>
    `,
    styles: [`这里省略一堆样式`]
    animations: [mask, dialog]
})
export class DialogComponent {
    public data: {
        title?: string,
        msg?: string,
        ok?: string,
        no?: string,
        mask?: string
    };
    public dialogRef: YupRef<DialogComponent>;
    public disp: string;
    constructor(
        private injector: Injector
    ) {
        this.data = this.injector.get(YUP_DATA);
        this.dialogRef = this.injector.get(YupRef);
        this.disp = 'init';
        setTimeout(() => {
            this.disp = 'on';
        });
    }
    public close(comfirm: boolean) {
        this.disp = 'off';
        setTimeout(() =>  {
            this.disp = 'init';
            this.dialogRef.close(comfirm);
        }, 300);
    }
}







public dialog(config: {
    title?: string,
    msg?: string,
    ok?: string,
    no?: string,
    mask?: boolean
}) {
    return this.open(DialogComponent, config);
}










import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { DialogComponent, AlertComponent, ToastComponent, LoadComponent } from './templates';
import { DialogService } from './service';
import { NoopAnimationsModule } from '@angular/platform-browser/animations';

@NgModule({
    declarations: [DialogComponent, AlertComponent, ToastComponent, LoadComponent],
    imports: [ NoopAnimationsModule, CommonModule ],
    exports: [],
    providers: [DialogService],
    entryComponents: [DialogComponent, AlertComponent, ToastComponent, LoadComponent]
})
export class YupModule {}




export { YupModule } from './module'; // 需要在AppModule中引入
export { DialogService as Yup } from './service'; // 用于发起弹窗
export { YupRef, YUP_DATA } from './popup.ref'; // 用于创建自定义弹窗时提供控制





constructor(
    public yup: Yup // 其实是DialogService，被笔者改了名
) { }

public ngOnInit() {
    this.yup.dialog({msg: '弹不弹?', title: '我弹', ok: '弹弹', no: '别弹了', mask: true}).afterClose().subscribe((res) => {
        if (res) {
            console.log('点击了确定');
        } else {
            console.log('点击了取消');
        }
    });
}
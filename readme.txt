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
    // ����һ����������ͨ�����ʹ���������ͨ��
    public open<T>(component: ComponentType<T>, config: any) {
        // �����������
        const factory = this.compFactRes.resolveComponentFactory(component);
        // ����һ���µĵ�������
        const dialogRef = new YupRef(factory, config);
        // �������õ��������(�ɵ������ô���������)append��body��ǩ��
        window.document.body.appendChild(this.getComponentRootNode(dialogRef.componentRef()));
        // ����angular����
        this.appRef.attachView(dialogRef.componentRef().hostView);
        // �������ĵ������÷��ظ����
        return dialogRef;
    }
    // �ο���Material2����ComponentRef���͵��������ת��ΪDOM�ڵ�
    private getComponentRootNode(componentRef: ComponentRef<any>): HTMLElement {
        return (componentRef.hostView as EmbeddedViewRef<any>).rootNodes[0] as HTMLElement;
    }
}

// �ο���Material2 ������Ϊ�������������
export interface ComponentType<T> {
    new (...args: any[]): T;
}











import { ComponentRef, InjectionToken, ReflectiveInjector, ComponentFactory } from '@angular/core';
import { Observable } from 'rxjs/Observable';
import { Subject } from 'rxjs/Subject';
// ����ע���Զ������ݵ������������
export const YUP_DATA = new InjectionToken<any>('YUPPopupData');

export class YupRef<T> {
    // �����رյĶ���
    private afterClose$: Subject<any>;
    // �������ñ���
    private dialogRef: ComponentRef<T>;
    constructor(
        private factory: ComponentFactory<T>,
        private config: any // ������Զ�������
    ) {
        this.afterClose$ = new Subject<any>();
        this.dialogRef = this.factory.create(
            ReflectiveInjector.resolveAndCreate([
                {provide: YUP_DATA, useValue: config}, // ע���Զ�������
                {provide: YupRef, useValue: this} // ע�����������Ϳ����ڴ�����������������Ĺرյ�
            ])
        );
    }
    // �ṩ�����ĶԴ��ڹرյĶ���
    public afterClose(): Observable<any> {
        return this.afterClose$.asObservable();
    }
    // �رշ��������������
    public close(data?: any) {
        this.afterClose$.next(data);
        this.afterClose$.complete();
        this.dialogRef.destroy();
    }
    // �ṩ�����������԰�����ӵ�DOM��
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
        <div class="yup-body-head">{{data?.title || '��Ϣ'}}</div>
        <div class="yup-body-content">{{data?.msg || ' '}}</div>
        <div class="yup-body-btns">
            <div class="btn default" (click)="close(false)">{{data?.no || 'ȡ��'}}</div>
            <div class="btn primary" (click)="close(true)">{{data?.ok || 'ȷ��'}}</div>
        </div>
    </div>
    `,
    styles: [`����ʡ��һ����ʽ`]
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




export { YupModule } from './module'; // ��Ҫ��AppModule������
export { DialogService as Yup } from './service'; // ���ڷ��𵯴�
export { YupRef, YUP_DATA } from './popup.ref'; // ���ڴ����Զ��嵯��ʱ�ṩ����





constructor(
    public yup: Yup // ��ʵ��DialogService�������߸�����
) { }

public ngOnInit() {
    this.yup.dialog({msg: '������?', title: '�ҵ�', ok: '����', no: '����', mask: true}).afterClose().subscribe((res) => {
        if (res) {
            console.log('�����ȷ��');
        } else {
            console.log('�����ȡ��');
        }
    });
}
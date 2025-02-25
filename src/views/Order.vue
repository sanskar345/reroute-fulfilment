<template>
  <ion-page>
    <ion-content :fullscreen="true">
      <div v-if="Object.keys(order).length">
        <ion-item class="ion-text-center" lines="none">
          <h1>{{ $t("Your Order") }}</h1>
        </ion-item>
        <ion-card>
          <ion-item lines="none" class="border">
            <ion-label>
              {{ order.customerName }}
              <p>{{ order.id }}</p>
            </ion-label>
            <ion-note slot="end">{{ $filters.formatDate(order.orderDate) }}</ion-note>
          </ion-item>
        </ion-card>
        <div v-if="order.statusId !== 'ORDER_CANCELLED' && order.shipGroup.length > 0" >
          <ion-card v-for="(shipGroup, index) of order.shipGroup" :key="index">
            <ion-item v-show="item.status !== 'ITEM_CANCELLED'" v-for="item of shipGroup.items" :key="item.id" lines="full">
              <ion-thumbnail slot="start">
                <Image :src='getProduct(item.productId).mainImageUrl' />
              </ion-thumbnail>
              <ion-label slot="start">
                <p>{{ item.brandName }}</p>
                <h2>{{ item.name }}</h2>
                <p v-for="(attribute, feature) in ($filters.groupFeatures(getProduct(item.productId).featureHierarchy))" :key="attribute" >
                  <span class="sentence-case">{{ feature }}</span>: {{ attribute }}
                </p>
              </ion-label>
            </ion-item>
            <!-- TODO -->
            <!-- <ion-item>
              <ion-label>
                status description
              </ion-label>
            </ion-item> -->
            <ion-item>
              <ion-label>{{ $t('Delivery method') }}</ion-label>
              <ion-select :disabled="!hasPermission(Actions.APP_SHPGRP_DLVRMTHD_UPDATE)" interface="popover" :value="shipGroup.selectedShipmentMethodTypeId" @ionChange="updateDeliveryMethod($event, shipGroup)">
                <ion-select-option v-for="method in deliveryMethods" :key="method.value" :value="method.value">{{ method.name }}</ion-select-option>
              </ion-select>
            </ion-item>
            <ion-button v-if="shipGroup.shipmentMethodTypeId === 'STOREPICKUP' && shipGroup.selectedShipmentMethodTypeId !== shipGroup.shipmentMethodTypeId && !shipGroup.updatedAddress" :disabled="!hasPermission(Actions.APP_SHPGRP_DLVRADR_UPDATE) && shipGroup.shipmentMethodTypeId !== 'STOREPICKUP'" @click="updateDeliveryAddress(shipGroup)" expand="block" fill="outline">{{ $t("Add address") }}</ion-button>
            <ion-button v-else-if="shipGroup.selectedShipmentMethodTypeId === 'STOREPICKUP' && !shipGroup.selectedFacility" :disabled="!hasPermission(Actions.APP_SHPGRP_PCKUP_UPDATE) && shipGroup.shipmentMethodTypeId === 'STOREPICKUP'" @click="updatePickupLocation(shipGroup)" expand="block" fill="outline">{{ $t("Select pickup location")}}</ion-button>
            <ion-item v-else-if="shipGroup.selectedShipmentMethodTypeId === 'STOREPICKUP'">
              <ion-list>
                <ion-label>{{ shipGroup.selectedFacility.facilityName }} </ion-label>
                <ion-label color="dark">{{ shipGroup.selectedFacility.address1 }} </ion-label>
                <ion-label color="dark">{{ shipGroup.selectedFacility.city }} {{ shipGroup.selectedFacility.stateCode }} {{ shipGroup.selectedFacility.postalCode }}</ion-label>
              </ion-list>
              <ion-button :disabled="!hasPermission(Actions.APP_SHPGRP_PCKUP_UPDATE) && shipGroup.shipmentMethodTypeId === 'STOREPICKUP'" slot="end" @click="updatePickupLocation(shipGroup)" color="medium" fill="outline">{{ $t("Change Store")}}</ion-button>
            </ion-item>
            <ion-item v-else>
              <ion-list v-if="shipGroup.updatedAddress">
                <ion-label>{{ shipGroup.updatedAddress.firstName }} {{ shipGroup.updatedAddress.lastName }}</ion-label>
                <ion-label color="dark">{{ shipGroup.updatedAddress.address1 }} </ion-label>
                <ion-label color="dark">{{ shipGroup.updatedAddress.city }} {{ shipGroup.updatedAddress.stateCode }} {{ shipGroup.updatedAddress.postalCode }}</ion-label>
              </ion-list>
              <ion-list v-else-if="shipGroup.shipmentMethodTypeId !== 'STOREPICKUP'">
                <ion-label>{{ shipGroup.shipTo.postalAddress.toName }}</ion-label>
                <ion-label color="dark">{{ shipGroup.shipTo.postalAddress.address1 }} </ion-label>
                <ion-label color="dark">{{ shipGroup.shipTo.postalAddress.city }} {{ shipGroup.shipTo.postalAddress.stateCode }} {{ shipGroup.shipTo.postalAddress.postalCode }}</ion-label>
              </ion-list>
              <ion-button :disabled="!hasPermission(Actions.APP_SHPGRP_DLVRADR_UPDATE) && shipGroup.shipmentMethodTypeId !== 'STOREPICKUP'" v-if="shipGroup.shipmentMethodTypeId !== 'STOREPICKUP' || shipGroup.updatedAddress" slot="end" @click="updateDeliveryAddress(shipGroup)" color="medium" fill="outline">{{ $t("Edit address") }}</ion-button>
            </ion-item>
            <!-- TODO -->
            <!-- <ion-item v-if="shipGroup.selectedShipmentMethodTypeId !== 'STOREPICKUP'" lines="none">
              <ion-label>{{ $t("Estimated delivery") }}</ion-label>
              <ion-label slot="end">{{ $t("03/03/2023") }}</ion-label>
            </ion-item> -->
            <ion-item v-if="shipGroup.trackingNumber">
              <ion-label>{{ $t('Tracking code') }}</ion-label>
              <ion-note slot="end">{{ shipGroup.trackingNumber }}</ion-note>
            </ion-item>
            <!-- Disabling the buttons if address or facility is not added -->
            <ion-button :disabled="(!shipGroup.updatedAddress && (!shipGroup.selectedFacility || shipGroup.selectedFacility.facilityId == shipGroup.facilityId))" @click="save(shipGroup)" fill="clear">{{ $t("Save changes") }}</ion-button>
            <ion-button :disabled="!hasPermission(Actions.APP_SHPGRP_CNCL)" @click="cancel(shipGroup)" fill="clear" color="danger">{{ $t("Cancel") }}</ion-button>
          </ion-card>
        </div>
        <div v-else class="ion-text-center ion-padding-top">
          <ion-label>{{ $t("Order item not eligible for reroute fulfilment") }}</ion-label>
        </div>
      </div>
      <div v-else-if="loader == null" class="ion-text-center ion-padding-top">
        <ion-label>{{ $t("Order not found") }}</ion-label>
      </div>
    </ion-content>
  </ion-page>
</template>

<script lang="ts">
import {
  alertController,
  IonButton,
  IonCard,
  IonContent,
  IonItem,
  IonLabel,
  IonList,
  IonNote,
  IonPage,
  IonSelect,
  IonSelectOption,
  IonThumbnail,
  loadingController,
  modalController,
} from "@ionic/vue";
import { defineComponent } from "vue";
import { useRouter } from "vue-router";
import { mapGetters, useStore } from 'vuex'
import { OrderService } from "@/services/OrderService";
import { translate } from "@/i18n";
import { hasError, showToast } from "@/utils";
import Image from "@/components/Image.vue";
import AddressModal from "@/views/AddressModal.vue";
import { ProductService } from "@/services/ProductService";
import PickupLocationModal from "@/views/PickupLocationModal.vue";
import { Actions, hasPermission } from '@/authorization'
import { initialise } from '@/adapter'

export default defineComponent({
  name: "Order",
  components: {
    Image,
    IonButton,
    IonCard,
    IonContent,
    IonItem,
    IonLabel,
    IonList,
    IonNote,
    IonSelect,
    IonSelectOption,
    IonThumbnail,
    IonPage,
  },
  data() {
    return {
      loader: null as any,
      maxAge: process.env.VUE_APP_CACHE_MAX_AGE ? parseInt(process.env.VUE_APP_CACHE_MAX_AGE) : 0,
      order: {} as any,
      products: {} as any,
      deliveryMethods: [
        {
          name: 'Store pickup',
          value: 'STOREPICKUP'
        },
        {
          name: 'Shipping',
          value: 'STANDARD'
        }
      ]
    }
  },
  computed: {
    ...mapGetters({
      deliveryMethod: 'user/getDeliveryMethod',
    })
  },
  async mounted() {
    if (Object.keys(this.$route.query).length > 0) {
      if(!this.$route.query.oms || !this.$route.query.token) {
        // invalid request
        return;
      }
      initialise({
        token: this.$route.query.token,
        instanceUrl: `${this.$route.query.oms}/api/`,
        cacheMaxAge: this.maxAge,
        events: {
          responseErrror: () => {
            setTimeout(() => this.dismissLoader(), 100);
          }
        }
      })
      await this.getOrder();
    }
  },
  methods: {
    async presentLoader() {
      this.loader = await loadingController
        .create({
          message: this.$t("Fetching order details."),
          translucent: true,
        });
      await this.loader.present();
    },
    dismissLoader() {
      if (this.loader) {
        this.loader.dismiss();
        this.loader = null;
      }
    },
    async getOrder() {
      await this.presentLoader()
      let resp;
      let order
      try {
        resp = await OrderService.getOrder();
        if (!hasError(resp)) {
          order = resp.data;
          const productIds: any = new Set();
          order.shipGroup = order.shipGroup.filter((group: any) => {
            if(group.facilityId === '_NA_') {
              group.selectedShipmentMethodTypeId = group.shipmentMethodTypeId;
              group.items = group.items.filter((item: any) => {
                if (item.status == 'ITEM_CANCELLED') return false;
                productIds.add(item.productId);
                return true;
              })
              return group.items.length > 0;
            }
          })
          if (productIds.length) await this.fetchProducts([...productIds])
          await this.store.dispatch("user/getConfiguration", order.productStoreId);
          this.order = order;
          if (productIds.size) await this.fetchProducts([...productIds])
        }
      } catch (error) {
        console.error(error)
      }
      this.dismissLoader()
    },
    async fetchProducts(productIds: any) {
      const productIdFilter = productIds.reduce((filter: string, productId: any) => {
        if (filter !== '') filter += ' OR '
        return filter += productId;
      }, '');

      try {
        const resp = await ProductService.fetchProducts({
          "filters": ['productId: (' + productIdFilter + ')'],
          "viewSize": productIds.length
        })

        if (resp.status === 200 && !hasError(resp) && resp.data) {
          resp.data.response.docs.forEach((product: any) => {
            this.products[product.productId] = product
          });
        }
      } catch (error) {
        console.error(error)
      }
    },

    getProduct(productId: string) {
      return this.products[productId] ? this.products[productId] : {}
    },

    async updateShippingAddress(shipGroup: any) {
      let resp
      const payload = {
        "orderId": this.order.id,
        "shipGroupSeqId": shipGroup.shipGroupSeqId,
        "contactMechId": shipGroup.shipmentMethodTypeId === 'STOREPICKUP' ? "" :shipGroup.shipTo.postalAddress.id,
        "shipmentMethod": `${this.deliveryMethod}@_NA_`,
        "contactMechPurposeTypeId": "SHIPPING_LOCATION",
        "facilityId": shipGroup.facilityId,
        "toName": `${shipGroup.updatedAddress.firstName} ${shipGroup.updatedAddress.lastName}`,
        "address1": shipGroup.updatedAddress.address1,
        "city": shipGroup.updatedAddress.city,
        "stateProvinceGeoId": shipGroup.updatedAddress.stateProvinceGeoId,
        "postalCode": shipGroup.updatedAddress.postalCode,
        "countryGeoId": shipGroup.updatedAddress.countryGeoId
      } as any

      if (shipGroup.selectedShipmentMethodTypeId === shipGroup.shipmentMethodTypeId) {
        // In case of address edit, we honour the previously selected delivery method
        payload.shipmentMethod = `${shipGroup.shipmentMethodTypeId}@_NA_`
        payload.isEdited = true
      }

      try {
        resp = await OrderService.updateShippingAddress(payload);
        if (resp.status === 200 && !hasError(resp) && resp.data) {
          shipGroup.shipTo.postalAddress = shipGroup.updatedAddress
          shipGroup.updatedAddress = null
          showToast(translate("Changes saved"))
        } else {
          showToast(translate("Failed to update the shipping addess"))
        }
      } catch (error) {
        console.error(error)
        showToast(translate("Failed to update the shipping addess"))
      }
      this.getOrder();
    },

    async updatePickupFacility(shipGroup: any) {
      let resp
      const payload = {
        "orderId": this.order.id,
        "shipGroupSeqId": shipGroup.shipGroupSeqId,
        "contactMechId": shipGroup.shipTo.postalAddress.id,
        "shipmentMethod": "STOREPICKUP@_NA_@CARRIER", // TODO Check why CARRIER is needed
        "contactMechPurposeTypeId": "SHIPPING_LOCATION",
        "facilityId": shipGroup.selectedFacility.facilityId,
      }

      try {
        resp = await OrderService.updatePickupFacility(payload);
        if (resp.status === 200 && !hasError(resp)) {
          shipGroup.facilityId = shipGroup.selectedFacility.facilityId
          showToast(translate("Changes saved"))
        } else {
          showToast(translate("Failed to update the pickup store"))
        }
      } catch (error) {
        console.error(error)
        showToast(translate("Failed to update the pickup store"))
      }
      this.getOrder();
    },

    updateDeliveryMethod(event: any, shipGroup: any) {
      const group = this.order.shipGroup.find((group: any) => group.shipGroupSeqId === shipGroup.shipGroupSeqId);
      group.selectedShipmentMethodTypeId = event.detail.value;
      // Resetting the previous changes on method change
      this.resetShipGroup(shipGroup)
    },

    async updateDeliveryAddress(shipGroup: any) {
      const modal = await modalController
        .create({
          component: AddressModal,
          // Adding backdropDismiss as false because on dismissing the modal through backdrop,
          // backrop.role returns 'backdrop' giving unexpected result
          backdropDismiss: false,
          componentProps: {
            shipGroup,
          }
        })
      modal.onDidDismiss().then((result) => {
        if (result.role) {
          // role will have the passed data
          shipGroup.updatedAddress = result.role
        }
      });
      return modal.present();
    },

    async updatePickupLocation(shipGroup: any) {
      const modal = await modalController
        .create({
          component: PickupLocationModal,
          // Adding backdropDismiss as false because on dismissing the modal through backdrop,
          // backrop.role returns 'backdrop' giving unexpected result
          backdropDismiss: false,
          componentProps: {
            shipGroup
          }
        })
      modal.onDidDismiss().then((result) => {
        if (result.role) {
          // role will have the passed data
          shipGroup.selectedFacility = result.role
        }
      });
      return modal.present();
    },

    async save(shipGroup: any) {
      const message = this.$t("Are you sure you want to save the changes?");
      const alert = await alertController.create({
        header: this.$t("Save changes"),
        message,
        buttons: [
          {
            text: this.$t("Cancel"),
          },
          {
            text: this.$t("Confirm"),
            handler: () => {
              shipGroup.selectedShipmentMethodTypeId === 'STOREPICKUP' ? this.updatePickupFacility(shipGroup) : this.updateShippingAddress(shipGroup);   
            }
          }
        ],
      });
      return alert.present();
    },


    async cancelShipGroup(shipGroup: any) {
      let resp
      const itemReasonMap = {} as any
      shipGroup.items.map((item: any) => itemReasonMap[item.itemSeqId] = 'OICR_CHANGE_MIND')
      const payload = {
        "orderId": this.order.id,
        "shipGroupSeqId": shipGroup.shipGroupSeqId,
        "itemReasonMap": itemReasonMap
      } as any

      try {
        resp = await OrderService.cancelOrderItem(payload);
        if (resp.status === 200 && !hasError(resp) && resp.data.orderId == this.order.id) {
          shipGroup.isCancelled = true;
          showToast(translate("Order cancelled successfully"))
        } else {
          showToast(translate("Failed to cancel the order"))
        }
      } catch (error) {
        console.error(error)
        showToast(translate("Failed~ to cancel the order"))
      }
      this.getOrder();
    },

    async cancel(shipGroup: any) {
      const message = this.$t("Are you sure you want to cancel the order items?");
      const alert = await alertController.create({
        header: this.$t("Cancel items"),
        message,
        buttons: [
          {
            text: this.$t("Don't Cancel"),
          },
          {
            text: this.$t("Cancel"),
            handler: () => {
              this.cancelShipGroup(shipGroup);
            }
          }
        ],
      });
      return alert.present();
    },

    resetShipGroup(shipGroup: any) {
      shipGroup.updatedAddress = null
      shipGroup.selectedFacility = null
    }
  },
  setup() {
    const router = useRouter();
    const store = useStore();
    return {
      Actions,
      hasPermission,
      router,
      store
    };
  }
});
</script>
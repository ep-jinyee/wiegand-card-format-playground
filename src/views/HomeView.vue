<script setup>
import Big from 'big.js';
import { ref, watch } from 'vue';

const E_WIEG_BIT_CARD_BIT = 1;
const E_WIEG_BIT_SITE_BIT = 2;
const E_WIEG_BIT_EVEN_PAR_MSK_1 = 4;
const E_WIEG_BIT_EVEN_PAR_MSK_2 = 8;
const E_WIEG_BIT_EVEN_PAR_MSK_3 = 16;
const E_WIEG_BIT_ODD_PAR_MSK_1 = 32;
const E_WIEG_BIT_ODD_PAR_MSK_2 = 64;
const E_WIEG_BIT_ODD_PAR_MSK_3 = 128;

const debug = ref(false);
const num_card_bit = ref(26);

const even_parity_bit_mask_pos = ref([[], [], []]);
const odd_parity_bit_mask_pos = ref([[], [], []]);

const cardsite_bit_pos_count_map_array = ref([]);
const card_bit_count_dustbin = ref([]);
const site_bit_count_dustbin = ref([]);
const card_bit_count = ref(0);
const site_bit_count = ref(0);

const card_bit_mouse_down = ref(0);
const site_bit_mouse_down = ref(0);
const e_par_1_bit_mouse_down = ref(0);
const e_par_2_bit_mouse_down = ref(0);
const e_par_3_bit_mouse_down = ref(0);
const o_par_1_bit_mouse_down = ref(0);
const o_par_2_bit_mouse_down = ref(0);
const o_par_3_bit_mouse_down = ref(0);

const sitecode_enabled = ref(false);

const num_even_parity = ref(0);
const num_odd_parity = ref(0);

const even_parity_pos = ref([]);
const odd_parity_pos = ref([]);

const num_sitecode_based_10_digits = ref(0);
const num_cardcode_based_10_digits = ref(0);

const max_num_sitecode_based_10_digits = ref(0);
const max_num_cardcode_based_10_digits = ref(0);

const max_sitecode_str = ref('0');
const max_cardcode_str = ref('0');

const card_format = ref({
    num_fc_digit: 0,
    num_card_digit: 0,
    convertion: 2,
    cardlen: num_card_bit.value,
    oddparity_pos: [],
    evenparity_pos: [],
    cardcode: []
});

const user_input_json_format = ref('');

function map_index_to_site_pos(i) {
    let ret = cardsite_bit_pos_count_map_array.value.find(w => w[0] == i && w[2] == 1)
    if (ret) {
        return ret[1] + "";
    }
    return "";
}

function map_index_to_card_pos(i) {
    let ret = cardsite_bit_pos_count_map_array.value.find(w => w[0] == i && w[2] == 0)
    if (ret) {
        return ret[1] + "";
    }
    return "";
}

function on_sitecode_cell_click(e, i) {
    console.log("mouse click")
    if (map_index_to_card_pos(i)) { // card code and site code are mutually exclusive
        return;
    }
    if (!map_index_to_site_pos(i)) {
        if (site_bit_count_dustbin.value.length) {
            cardsite_bit_pos_count_map_array.value.push([i, site_bit_count_dustbin.value.pop(), 1]);
        } else {
            cardsite_bit_pos_count_map_array.value.push([i, ++site_bit_count.value, 1]);
        }
    } else {
        cardsite_bit_pos_count_map_array.value = cardsite_bit_pos_count_map_array.value.filter(w => {
            if (w[0] == i && w[2] == 1) {
                site_bit_count_dustbin.value.push(w[1]);
                site_bit_count_dustbin.value.sort((a, b) => b - a);
                return false;
            }
            return true;
        });
    }
}

function on_sitecode_cell_mousedown(e, i) {
    site_bit_mouse_down.value = 1;
}

function on_sitecode_cell_mouseup(e, i) {
    on_sitecode_cell_click(null, i);
    site_bit_mouse_down.value = 0;
}

function on_sitecode_cell_mouseleave(e, last) {
    if (!site_bit_mouse_down.value || map_index_to_site_pos(last)) {
        return;
    }

    on_sitecode_cell_click(null, last);
}

function on_cardcode_cell_click(e, i) {
    if (map_index_to_site_pos(i)) { // card code and site code are mutually exclusive
        return;
    }
    if (!map_index_to_card_pos(i)) {
        if (card_bit_count_dustbin.value.length) {
            cardsite_bit_pos_count_map_array.value.push([i, card_bit_count_dustbin.value.pop(), 0]);
        } else {
            cardsite_bit_pos_count_map_array.value.push([i, ++card_bit_count.value, 0]);
        }
    } else {
        cardsite_bit_pos_count_map_array.value = cardsite_bit_pos_count_map_array.value.filter(w => {
            if (w[0] == i && w[2] == 0) {
                card_bit_count_dustbin.value.push(w[1]);
                card_bit_count_dustbin.value.sort((a, b) => b - a);
                return false;
            }
            return true;
        });
    }
}

function on_cardcode_cell_mousedown(e, i) {

    card_bit_mouse_down.value = 1;
}

function on_cardcode_cell_mouseup(e, i) {

    on_cardcode_cell_click(null, i);
    card_bit_mouse_down.value = 0;
}

function on_cardcode_cell_mouseleave(e, last) {
    if (!card_bit_mouse_down.value || map_index_to_card_pos(last)) {
        return;
    }
    on_cardcode_cell_click(null, last);
}

function on_even_parity_cell_mousedown(e, i, j) {
    if (j == 1) e_par_1_bit_mouse_down.value = 1;
    if (j == 2) e_par_2_bit_mouse_down.value = 1;
    if (j == 3) e_par_3_bit_mouse_down.value = 1;
}

function on_even_parity_cell_mouseup(e, i, j) {
    add_remove_even_parity_bit(e, i, j);
    if (j == 1) e_par_1_bit_mouse_down.value = 0;
    if (j == 2) e_par_2_bit_mouse_down.value = 0;
    if (j == 3) e_par_3_bit_mouse_down.value = 0;
}

function on_even_parity_cell_mouseleave(e, i, j) {
    if (j == 1 && e_par_1_bit_mouse_down.value) add_remove_even_parity_bit(e, i, j);
    if (j == 2 && e_par_2_bit_mouse_down.value) add_remove_even_parity_bit(e, i, j);
    if (j == 3 && e_par_3_bit_mouse_down.value) add_remove_even_parity_bit(e, i, j);
}

function on_odd_parity_cell_mousedown(e, i, j) {
    if (j == 1) o_par_1_bit_mouse_down.value = 1;
    if (j == 2) o_par_2_bit_mouse_down.value = 1;
    if (j == 3) o_par_3_bit_mouse_down.value = 1;
}

function on_odd_parity_cell_mouseup(e, i, j) {
    add_remove_odd_parity_bit(e, i, j);
    if (j == 1) o_par_1_bit_mouse_down.value = 0;
    if (j == 2) o_par_2_bit_mouse_down.value = 0;
    if (j == 3) o_par_3_bit_mouse_down.value = 0;
}

function on_odd_parity_cell_mouseleave(e, i, j) {
    if (j == 1 && o_par_1_bit_mouse_down.value) add_remove_odd_parity_bit(e, i, j);
    if (j == 2 && o_par_2_bit_mouse_down.value) add_remove_odd_parity_bit(e, i, j);
    if (j == 3 && o_par_3_bit_mouse_down.value) add_remove_odd_parity_bit(e, i, j);
}

function add_remove_even_parity_bit(e, i, j) {
    if (even_parity_bit_mask_pos.value[j - 1].includes(i)) {
        even_parity_bit_mask_pos.value[j - 1] = [...even_parity_bit_mask_pos.value[j - 1].filter(k => k != i)]
        return;
    }

    if (even_parity_pos.value[j - 1] == i) {
        // even_parity_bit_mask_pos.value[j - 1] = [...even_parity_bit_mask_pos.value[j - 1].filter(k => k != i)]
        return;
    }

    even_parity_bit_mask_pos.value[j - 1].push(i);
}

function add_remove_odd_parity_bit(e, i, j) {
    console.log("What is i , j ", i, j)
    if (odd_parity_bit_mask_pos.value[j - 1].includes(i)) {
        odd_parity_bit_mask_pos.value[j - 1] = [...odd_parity_bit_mask_pos.value[j - 1].filter(k => k != i)]
        return;
    }

    if (odd_parity_pos.value[j - 1] == i) {
        console.log("ANBU")
        // odd_parity_bit_mask_pos.value[j - 1] = [...odd_parity_bit_mask_pos.value[j - 1].filter(k => k != i)]
        return;
    }

    odd_parity_bit_mask_pos.value[j - 1].push(i);
}

function on_reset_pressed(e) {
    num_card_bit.value = 0;
    num_even_parity.value = 0;
    num_odd_parity.value = 0;
    sitecode_enabled.value = false;
    even_parity_pos.value = [];
    odd_parity_pos.value = [];
    card_bit_count.value = 0;
    site_bit_count.value = 0;
    card_bit_count_dustbin.value = [];
    site_bit_count_dustbin.value = [];
    cardsite_bit_pos_count_map_array.value = [];
    even_parity_bit_mask_pos.value = [[], [], []];
    odd_parity_bit_mask_pos.value = [[], [], []];
    user_input_json_format.value = '';
    num_cardcode_based_10_digits.value = 0;
    num_sitecode_based_10_digits.value = 0;
    max_num_cardcode_based_10_digits.value = 0;
    max_cardcode_str.value = '0';
    max_num_sitecode_based_10_digits.value = 0;
    max_sitecode_str.value = '0';
    card_format.value = {
        convertion: 2,
        cardlen: num_card_bit.value,
        oddparity_pos: [],
        evenparity_pos: [],
        cardcode: []
    };
}

function generate_card_format_json(e) {

    card_format.value.evenparity_pos = even_parity_pos.value;
    card_format.value.oddparity_pos = odd_parity_pos.value;

    console.log(even_parity_bit_mask_pos)
    for (let i = 0; i < 3; i++)
    {   
        if (!even_parity_pos.value[i]) {
            card_format.value.evenparity_pos[i] = 255
        }
        if (!odd_parity_pos.value[i]) {
            card_format.value.oddparity_pos[i] = 255
        }
    }

    card_format.value.cardcode = [];
    card_format.value.cardlen = card_bit_count.value;

    // no site code, just CSN
    if (card_format.value.convertion == 2) 
    {
        card_format.value.num_card_digit = 0,
        card_format.value.num_fc_digit = 0
    } else {
        card_format.value.num_card_digit = num_cardcode_based_10_digits.value;
        card_format.value.num_fc_digit = num_sitecode_based_10_digits.value;
    }


    // todo :: validate first
    for (let i = 1; i <= num_card_bit.value; i++) {
        let type = 0;
        let pos = 0;
        for (let j = 1; j <= 3; j++) {
            if (even_parity_bit_mask_pos.value[j - 1].includes(i)) {
                if (j == 1) type += E_WIEG_BIT_EVEN_PAR_MSK_1;
                if (j == 2) type += E_WIEG_BIT_EVEN_PAR_MSK_2;
                if (j == 3) type += E_WIEG_BIT_EVEN_PAR_MSK_3;
            }

            if (odd_parity_bit_mask_pos.value[j - 1].includes(i)) {
                if (j == 1) type += E_WIEG_BIT_ODD_PAR_MSK_1;
                if (j == 2) type += E_WIEG_BIT_ODD_PAR_MSK_2;
                if (j == 3) type += E_WIEG_BIT_ODD_PAR_MSK_3;
            }
        }
        // YOU WILL ONLY TAKE THE FIRST PARITY BIT IF YOU DO LIKE THIS BELOW
        // let parityIndex = even_parity_bit_mask_pos.value.findIndex(a => a.includes(i));
        // if (parityIndex != -1) {
        //     if (parityIndex == 0) type += E_WIEG_BIT_EVEN_PAR_MSK_1;
        //     else if (parityIndex == 1) type += E_WIEG_BIT_EVEN_PAR_MSK_2;
        //     else if (parityIndex == 2) type += E_WIEG_BIT_EVEN_PAR_MSK_3;
        // }
        // parityIndex = odd_parity_bit_mask_pos.value.findIndex(a => a.includes(i));
        // if (parityIndex != -1) {
        //     if (parityIndex == 0) type += E_WIEG_BIT_ODD_PAR_MSK_1;
        //     else if (parityIndex == 1) type += E_WIEG_BIT_ODD_PAR_MSK_2;
        //     else if (parityIndex == 2) type += E_WIEG_BIT_ODD_PAR_MSK_3;
        // }
        let card_or_site_bit = cardsite_bit_pos_count_map_array.value.find(c => c[0] == i);
        if (card_or_site_bit) {
            if (card_or_site_bit[2] == 0) { // card bit
                type += E_WIEG_BIT_CARD_BIT;
                pos += card_or_site_bit[1] - 1;
            }
            if (card_or_site_bit[2] == 1) { // site bit
                type += E_WIEG_BIT_SITE_BIT;
                pos += card_or_site_bit[1] - 1;
            }
        }
        if (type != 0 || pos != 0) {
            card_format.value.cardcode.push((type << 8) | pos);
        } else {
            card_format.value.cardcode.push(0);
        }
    }
    card_format.value.cardlen = num_card_bit.value;
    user_input_json_format.value = JSON.stringify(card_format.value, null, 2);
}

function on_even_parity_pos_change(e, j) {
    // regardless odd or even, parity codes are mutually exclusive
    let eindex = even_parity_pos.value.findIndex(i => i == Number(e.target.value));
    let oindex = odd_parity_pos.value.findIndex(i => i == Number(e.target.value));
    if (eindex != -1 || oindex != -1) {
        if (eindex != -1)
            even_parity_pos.value.splice(eindex, 1);
        if (oindex != -1)
            odd_parity_pos.value.splice(eindex, 1);
    }

    // parity bit and its mask bits are also mutually exclusive
    if (even_parity_bit_mask_pos.value[j - 1].includes(Number(e.target.value))) {
        even_parity_bit_mask_pos.value[j - 1] = [...even_parity_bit_mask_pos.value[j - 1].filter(k => k != Number(e.target.value))];
        even_parity_pos.value[j - 1] = Number(e.target.value);
    }

    even_parity_pos.value[j - 1] = Number(e.target.value);
}

function on_odd_parity_pos_change(e, j) {
    // regardless odd or even, parity codes are mutually exclusive
    let eindex = even_parity_pos.value.findIndex(i => i == Number(e.target.value));
    let oindex = odd_parity_pos.value.findIndex(i => i == Number(e.target.value));
    if (eindex != -1 || oindex != -1) {
        if (eindex != -1)
            even_parity_pos.value.splice(eindex, 1);
        if (oindex != -1)
            odd_parity_pos.value.splice(eindex, 1);
    }

    if (odd_parity_bit_mask_pos.value[j - 1].includes(Number(e.target.value))) {
        odd_parity_bit_mask_pos.value[j - 1] = [...odd_parity_bit_mask_pos.value[j - 1].filter(k => k != Number(e.target.value))];
        odd_parity_pos.value[j - 1] = Number(e.target.value);
    }

    odd_parity_pos.value[j - 1] = Number(e.target.value);
}

function on_site_code_click() {
    sitecode_enabled.value = !sitecode_enabled.value;
    if (sitecode_enabled.value) {
        card_format.value['convertion'] = 1;
    } else {
        card_format.value['convertion'] = 2;
    }
}

function on_user_change_json_format() {
    try {
        let json = JSON.parse(user_input_json_format.value);

        if (json.cardlen) {
            num_card_bit.value = json.cardlen;
        }

        if (json.convertion) {
            card_format.value.convertion = json.convertion;
        }

        if (json.oddparity_pos) {
            odd_parity_pos.value = json.oddparity_pos;
            card_format.value.oddparity_pos = json.oddparity_pos;
            num_odd_parity.value = json.oddparity_pos.filter(a => a != 255).length
        }

        if (json.evenparity_pos) {
            even_parity_pos.value = json.evenparity_pos;
            card_format.value.evenparity_pos = json.evenparity_pos;
            num_even_parity.value = json.evenparity_pos.filter(a => a != 255).length
        }

        if (json.cardcode) {
            card_format.value.cardcode = json.cardcode;
        }

        // to clear the existing card format
        cardsite_bit_pos_count_map_array.value = [];
        even_parity_bit_mask_pos.value = [[],[],[]];
        odd_parity_bit_mask_pos.value = [[],[],[]];

        // need to reverse engineer the card format and draw the card format on table
        for (let i = 0; i < json.cardcode.length; i++) {
            let type = json.cardcode[i] >> 8;
            let pos = json.cardcode[i] & 0xff;
            if (type & E_WIEG_BIT_CARD_BIT) {
                card_bit_count.value = Math.max(card_bit_count.value, pos + 1);
                cardsite_bit_pos_count_map_array.value.push([i + 1, pos + 1, 0]);
            }
            if (type & E_WIEG_BIT_SITE_BIT) {
                site_bit_count.value = Math.max(site_bit_count.value, pos + 1);
                cardsite_bit_pos_count_map_array.value.push([i + 1, pos + 1, 1]);
            }
            if (type & E_WIEG_BIT_EVEN_PAR_MSK_1) {
                even_parity_bit_mask_pos.value[0].push(i + 1);
            }
            if (type & E_WIEG_BIT_EVEN_PAR_MSK_2) {
                even_parity_bit_mask_pos.value[1].push(i + 1);
            }
            if (type & E_WIEG_BIT_EVEN_PAR_MSK_3) {
                even_parity_bit_mask_pos.value[2].push(i + 1);
            }
            if (type & E_WIEG_BIT_ODD_PAR_MSK_1) {
                odd_parity_bit_mask_pos.value[0].push(i + 1);
            }
            if (type & E_WIEG_BIT_ODD_PAR_MSK_2) {
                odd_parity_bit_mask_pos.value[1].push(i + 1);
            }
            if (type & E_WIEG_BIT_ODD_PAR_MSK_3) {
                odd_parity_bit_mask_pos.value[2].push(i + 1);
            }
        }

        if (site_bit_count.value > 0) {
            sitecode_enabled.value = true;
        }

        num_sitecode_based_10_digits.value = json.num_fc_digit || 0;
        num_cardcode_based_10_digits.value = json.num_card_digit || 0;
    } catch (e) {
        console.log(e);
    }
}

watch(() => site_bit_count.value, (newVal, oldVal) => {
    let x = new Big(2);
    let y = x.pow(newVal);
    max_sitecode_str.value = y.minus(1).toString();
    max_num_sitecode_based_10_digits.value = y.minus(1).toString().length;
});

watch(() => card_bit_count.value, (newVal, oldVal) => {
    let x = new Big(2);
    let y = x.pow(newVal);
    max_cardcode_str.value = y.minus(1).toString();
    max_num_cardcode_based_10_digits.value = y.minus(1).toString().length;
});


</script>

<template>
    <div style=" margin-bottom: 1rem">
        <div>
            <label>Enter number of card bit:</label>
            <input type="number" v-model="num_card_bit" />
        </div>
        <div>
            <label>Enable facility code</label>
            <input type="checkbox" @click="on_site_code_click" v-model="sitecode_enabled" />
        </div>
        <div v-if="sitecode_enabled">
            <div>
                <label>Number of Facility Code Digits (Based-10)</label>
                <input type="number" v-model="num_sitecode_based_10_digits" />
            </div>
            <div>
                <label>Number of Card Code Digits (Based-10)</label>
                <input type="number" v-model="num_cardcode_based_10_digits" />
            </div>
        </div>

    </div>

    <div style="margin-bottom: 1rem;">
        <div>
            <label>Add even parity code</label>
            <button @click="num_even_parity--" :disabled="num_even_parity == 0">-</button>
            <button @click="num_even_parity++" :disabled="num_even_parity == 3">+</button>
            <div v-for="i of num_even_parity">
                <label>Even Parity code {{ i }} position</label>
                <input type="number" @input="(e) => on_even_parity_pos_change(e, i)" :value="even_parity_pos[i - 1]" />
            </div>
        </div>
        <div>
            <label>Add odd parity code</label>
            <button @click="num_odd_parity--" :disabled="num_odd_parity == 0">-</button>
            <button @click="num_odd_parity++" :disabled="num_odd_parity == 3">+</button>
            <div v-for="i of num_odd_parity">
                <label>Odd parity code {{ i }} position</label>
                <input type="number" @input="(e) => on_odd_parity_pos_change(e, i)" :value="odd_parity_pos[i - 1]" />
            </div>
        </div>
    </div>

    <div style="margin-bottom: 1rem;">
        <button
            style="padding: 1rem; background-color: rgb(249,149,149); border: 0; box-shadow: 1px 1px 1px rgba(140,140,140); border-radius: 10px; display:inline-block; margin-right:10px"
            @click="on_reset_pressed">Reset</button>
        <button
            style="padding: 1rem; background-color: rgb(149,249,149); border: 0; box-shadow: 1px 1px 1px rgba(140,140,140); border-radius: 10px; display:inline-block; margin-right:10px"
            @click="generate_card_format_json">Generate Wiegand Format JSON</button>
        <label>Debug mode</label>
        <input type="checkbox" v-model="debug" />
    </div>

    <div>
        For convertion type,
        <ul>
            <li>0 - undefined</li>
            <li>1 - site code plus card code. For Edge, the format is site code + card code, whereby only maximum 15
                digit site code plus maximum 25 digit card code, total 40 digits</li>
            <li>2 - card code only. For Edge, 16 bytes or max 39 digit</li>
        </ul>
        Program will automatically change convertion type to 1 if site code is enabled.
    </div>

    <table>
        <thead>
            <tr style="height: 2rem">
                <th><span style="font-weight:bold">Bits:</span></th>
                <template v-for="i of num_card_bit">
                    <th style="width: 1.5rem"><span>{{ i }}</span></th>
                </template>
            </tr>
        </thead>
        <tbody>
            <tr v-if="sitecode_enabled">
                <td>Site code<span style="font-weight:bold">:</span></td>
                <template v-for="i of num_card_bit">
                    <td class="bg-site-bit" :class="{
                        active: map_index_to_site_pos(i)
                    }" style="width: 1.5rem" @mousedown="e => on_sitecode_cell_mousedown(e, i)"
                        @mouseup="(e) => on_sitecode_cell_mouseup(e, i)"
                        @mouseleave="(e) => on_sitecode_cell_mouseleave(e, i)">
                        <span>
                            {{ map_index_to_site_pos(i) }}
                        </span>
                    </td>
                </template>
            </tr>
            <tr>
                <td>Card bit<span style="font-weight:bold">:</span></td>
                <template v-for="i of num_card_bit">
                    <td class="bg-card-bit" :class="{ active: map_index_to_card_pos(i) }" style="width: 1.5rem"
                        @mouseleave="(e) => on_cardcode_cell_mouseleave(e, i)"
                        @mousedown="(e) => on_cardcode_cell_mousedown(e, i)"
                        @mouseup="(e) => on_cardcode_cell_mouseup(e, i)">
                        <span>{{ map_index_to_card_pos(i) }}</span>
                    </td>
                </template>
            </tr>
            <tr v-if="num_even_parity" v-for="j of num_even_parity">
                <td>Even Parity {{ j }}<span>:</span></td>
                <template v-for="i of num_card_bit">
                    <td v-if="even_parity_pos[j - 1] != null && i == even_parity_pos[j - 1]">
                        *
                    </td>
                    <td v-else
                        :class="{ 'bg-even-parity-1': (j == 1), 'bg-even-parity-2': (j == 2), 'bg-even-parity-3': (j == 3), 'active': even_parity_bit_mask_pos[j - 1].includes(i) }"
                        style="width:1.5rem" @mousedown="on_even_parity_cell_mousedown(e, i, j)"
                        @mouseleave="on_even_parity_cell_mouseleave(e, i, j)"
                        @mouseup="on_even_parity_cell_mouseup(e, i, j)"></td>
                </template>
            </tr>
            <tr v-if="num_odd_parity" v-for="j of num_odd_parity">
                <td>Odd Parity {{ j }}<span style="font-weight:bold">:</span></td>
                <template v-for="i of num_card_bit">
                    <td v-if="odd_parity_pos[j - 1] != null && i == odd_parity_pos[j - 1]">
                        *
                    </td>
                    <td v-else
                        :class="{ 'bg-odd-parity-1': (j == 1), 'bg-odd-parity-2': (j == 2), 'bg-odd-parity-3': (j == 3), 'active': odd_parity_bit_mask_pos[j - 1].includes(i) }"
                        @mousedown="on_odd_parity_cell_mousedown(e, i, j)"
                        @mouseleave="on_odd_parity_cell_mouseleave(e, i, j)"
                        @mouseup="on_odd_parity_cell_mouseup(e, i, j)" style="width:1.5rem"></td>
                </template>
            </tr>
        </tbody>
    </table>
    <div v-if="sitecode_enabled">
        <p>With {{ site_bit_count }} site bits, the site code can be range from 0 up to {{ max_sitecode_str }}, its
            number of
            based-10
            digits of can be range from 0 up to {{
                max_num_sitecode_based_10_digits }}, user choose
            to
            have {{ num_sitecode_based_10_digits }} digits of site bits.</p>
        <p>With {{ card_bit_count }} card bits, the card code can be range from 0 up to {{ max_cardcode_str }}, its
            number of based-10
            digits can be range from 0 up to {{
                max_num_cardcode_based_10_digits }}, user choose
            to have {{ num_cardcode_based_10_digits }} digits of card bits.</p>
        <p>e.g. If site code = 1 and card code = 5, the resulting formatted card string under this format will be {{ '1' + '5'.padStart(num_cardcode_based_10_digits,'0') }}</p>
    </div>
    <div style="display:flex; margin-bottom: 1rem; flex-direction: row;">
        <div style="flex: 1 1 50%;">
            <div>
                <p>Card format</p>
                <pre>{{ card_format }}</pre>
            </div>
            <div v-if="debug">
                <p>Even parity bit mask position</p>
                <pre>{{ even_parity_bit_mask_pos }}</pre>
                <p>Odd Parity bit mask position</p>
                <pre>{{ odd_parity_bit_mask_pos }}</pre>
                <p>Even Parity Position</p>
                <pre>{{ even_parity_pos }}</pre>
                <p>Odd Parity Position</p>
                <pre>{{ odd_parity_pos }}</pre>
                <p>Card bit array is:</p>
                <pre>{{ cardsite_bit_pos_count_map_array }}</pre>
            </div>
        </div>
        <div style="flex: 2 0 70%;">
            <p>You can also put your format and generate the design. </p>
            <p>For example you can copy the following default settings in EdgeLPR board and paste into the textarea to visualize their format:</p>
            <ol>
                <li>26-bit Open Wiegand Format</li>
                <pre>
{
    "num_fc_digit": 4,
    "num_card_digit": 6,
    "convertion": 1,
    "cardlen": 26,
    "oddparity_pos": [26, 255, 255],
    "evenparity_pos": [1, 255, 255],
    "cardcode": [
        0, 1536, 1537, 1538, 1539, 1540, 1541, 1542,
        1543, 1280, 1281, 1282, 1283, 8452, 8453, 8454, 
        8455, 8456, 8457, 8458, 8459, 8460, 8461, 8462, 
        8463, 0
    ]
}
                </pre>
                <li>32-bit - 4BYTE CSN 32BIT</li>
                <pre>
{
  "num_fc_digit": 0,
  "num_card_digit": 0,
  "convertion": 2,
  "cardlen": 32,
  "oddparity_pos": [255, 255, 255],
  "evenparity_pos": [255, 255, 255],
  "cardcode": [
    256, 257, 258, 259, 260, 261, 262, 263, 
    264, 265, 266, 267, 268, 269, 270, 271,
    272, 273, 274, 275, 276, 277, 278, 279,
    280, 281, 282, 283, 284, 285, 286, 287
  ]
}
                </pre>
                <li>HID STANDARD 34BIT - (H10306)</li>
                <pre>
{
  "num_fc_digit": 5,
  "num_card_digit": 5,
  "convertion": 1,
  "cardlen": 34,
  "oddparity_pos": [34,255,255],
  "evenparity_pos": [1,255,255],
  "cardcode": [ 
    0, 1536, 1537, 1538, 1539, 1540, 1541, 1542, 
    1543, 1544, 1545, 1546, 1547, 1548, 1549, 1550,
    1551, 8448, 8449, 8450, 8451, 8452, 8453, 8454,
    8455, 8456, 8457, 8458, 8459, 8460, 8461, 8462,
    8463, 0
  ]
}
                </pre>
                <li>HID 35BIT CORPORATE 1000</li>
                <pre>
{
  "convertion": 1,
  "cardlen": 35,
  "oddparity_pos": [35,1,255],
  "evenparity_pos": [2,255,255],
  "cardcode": [
    0,24576,26112,17921,25090,26115,17924,25093,
    26118,17927,25096,26121,17930,25099,25856,17665,
    24834,25859,17668,24837,25862,17671,24840,25865,
    17674,24843,25868,17677,24846,25871,17680,24849,
    25874,17683,16384],
  "num_card_digit": 7,
  "num_fc_digit": 4
}
                </pre>
                <li>37 Bit - HID FARPOINTE 37BIT WITH SITE CODE - (H10304)</li>
                <pre>
{
  "num_fc_digit": 5,
  "num_card_digit": 6,
  "convertion": 1,
  "cardlen": 37,
  "oddparity_pos": [37,255,255],
  "evenparity_pos": [1,255,255],
  "cardcode": [
    0,1536,1537,1538,1539,1540,1541,1542,
    1543,1544,1545,1546,1547,1548,1549,1550,
    1551,1280,9473,8450,8451,8452,8453,8454,
    8455,8456,8457,8458,8459,8460,8461,8462,
    8463,8464,8465,8466,0
  ]
}
                </pre>
                <li>64 BIT CSN</li>
                <pre>
{
  "convertion": 2,
  "cardlen": 64,
  "oddparity_pos": [255,255,255],
  "evenparity_pos": [255,255,255],
  "cardcode": [
    256,257,258,259,260,261,262,263,
    264,265,266,267,268,269,270,271,
    272,273,274,275,276,277,278,279,
    280,281,282,283,284,285,286,287,
    288,289,290,291,292,293,294,295,
    296,297,298,299,300,301,302,303,
    304,305,306,307,308,309,310,311,
    312,313,314,315,316,317,318,319
  ],
  "num_card_digit": 0,
  "num_fc_digit": 0
}
                </pre>
                <li>92 BIT CSN</li>
                <pre>
{
  "convertion": 2,
  "cardlen": 92,
  "oddparity_pos": [255, 255, 255],
  "evenparity_pos": [255, 255, 255],
  "cardcode": [
    256,257,258,259,260,261,262,263,
    264,265,266,267,268,269,270,271,
    272,273,274,275,276,277,278,279,
    280,281,282,283,284,285,286,287,
    288,289,290,291,292,293,294,295,
    296,297,298,299,300,301,302,303,
    304,305,306,307,308,309,310,311,
    312,313,314,315,316,317,318,319,
    320,321,322,323,324,325,326,327,
    328,329,330,331,332,333,334,335,
    336,337,338,339,340,341,342,343,
    344,345,346,347
  ],
  "num_card_digit": 0,
  "num_fc_digit": 0
}
                </pre>
            </ol>

            <textarea style="width: 90%; height: 100px" v-model="user_input_json_format"
                @change="on_user_change_json_format"></textarea>
        </div>
    </div>
</template>

<style lang="scss" scoped>
table,
th,
td {
    border: 1px solid #4CAF50;
    transition: background-color 0.1s ease-in-out;
    user-select: none;
}

tr:nth-child(odd) td:nth-child(1) {
    background-color: #4682b4;
    color: white;
}

td {
    height: 5rem;
}

table {
    margin-top: 1rem;
}

td.bg-card-bit:hover,
td.bg-card-bit.active {
    background-color: #FFE8AA
}

td.bg-site-bit:hover,
td.bg-site-bit.active {
    background-color: #FFCBAA;
}

td.bg-even-parity-1:hover,
td.bg-even-parity-1.active {
    background-color: #98fb98;
}

td.bg-even-parity-2:hover,
td.bg-even-parity-2.active {
    background-color: #66cdaa;
}

td.bg-even-parity-3:hover,
td.bg-even-parity-3.active {
    background-color: #3cb371;
}

td.bg-odd-parity-1:hover,
td.bg-odd-parity-1.active {
    background-color: #Add8E6;
}

td.bg-odd-parity-2:hover,
td.bg-odd-parity-2.active {
    background-color: #87ceeb;
}

td.bg-odd-parity-3:hover,
td.bg-odd-parity-3.active {
    background-color: #4682b4;
}
</style>
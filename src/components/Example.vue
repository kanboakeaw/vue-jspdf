<template>
  <div id="Example">
      <div>
        <button @click="genaratePdf">genarate pdf</button>
        <button @click="genarateHTML">genarate pdf with HTML2Canvas</button>
        <button @click="genarateImage">HTML to image</button>
      </div>
      <div style="display:flex;">
           <!-- HTML2Canvas -->
        <div ref="pdfCanvas1" style="width: fit-content;height: fit-content;">
            <img alt="Vue logo" src="@/assets/logo.png">
            <h1>PDF with HTML2Canvas</h1>
            <HelloWorld msg="Welcome to Your Vue.js App"/>
        </div>
         <!-- Image -->
        <div style="display:flex;flex-direction:column;border: 1px solid #ddd;border-radius: 4px;">
            <span>Image from HTML2Canvas</span>
            <img ref="imgCanvas">
        </div>
      </div>

  </div>
</template>

<script>
import jsPDF from "jspdf";
import html2canvas from 'html2canvas';
import HelloWorld from '@/components/HelloWorld.vue'

export default {
    name : 'PdfExample',
    components: {
      HelloWorld
    },
    data(){
        return{
            // for jsPDF : https://rawgit.com/MrRio/jsPDF/master/docs/jsPDF.html
            pdfOption: {
                orientation: "p",
                format: "a4",
                unit: "mm",
                lineHeight: 1,
                putOnlyUsedFonts : true
            },
            pdfConfig: {
                margin: {
                    t: 15,
                    r: 20,
                    b: 15,
                    l: 20
                },
                typo: {
                    header: 20,
                    large: 18,
                    normal: 16,
                    small: 10
                },
                lineHeight: {
                    header: 2.5,
                    large: 1.75,
                    normal: 1.25,
                    small: 1.25
                }
            },
            canvasOption: {
                scale: 2,
                useCORS: true,
                logging: false,
                backgroundColor: '#ffffff'
            }
        }
    },
    methods:{
        drawParagraphWithIndent(pdf, text, x, y, maxWidth, indent = 0, lineHeightFactor = this.pdfConfig.lineHeight.normal, fontSize = this.pdfConfig.typo.normal) {
            const paragraphText = (text ?? '').toString();
            if (!paragraphText.trim()) {
            return y;
            }

            const firstLineWidth = Math.max(1, maxWidth - indent);
            const firstLineSplit = pdf.splitTextToSize(paragraphText, firstLineWidth);
            const firstLine = firstLineSplit[0] || '';
            const scaleFactor = pdf.internal.scaleFactor || 1;
            const lineHeight = (fontSize * lineHeightFactor) / scaleFactor;

            pdf.text(firstLine, x + indent, y);

            const remainingText = paragraphText.slice(firstLine.length).trimStart();
            if (!remainingText) {
            return y + lineHeight;
            }

            const remainingLines = pdf.splitTextToSize(remainingText, maxWidth);
            const nextY = y + lineHeight;
            pdf.text(remainingLines, x, nextY, { lineHeightFactor });

            return nextY + (remainingLines.length * lineHeight);
        },
        genaratePdf() {
            try {
                let pdf = new jsPDF(this.pdfOption);

                const pdf_width = pdf.internal.pageSize.width;
                const pdf_height = pdf.internal.pageSize.height;
                const margin_l = this.pdfConfig.margin.l;
                const margin_r = this.pdfConfig.margin.r;

                let pdf_position_y = this.pdfConfig.margin.t;

                pdf.setFont('Prompt','normal');
                pdf.setFontSize(this.pdfConfig.typo.header);
                pdf.setTextColor('#005D8E');
                pdf.text('sfsfdfsd sdsdgsdg', pdf_width / 2, pdf_position_y, null, null, "center");
                pdf_position_y += this.pdfConfig.typo.header * this.pdfConfig.lineHeight.header;
                
                pdf.setFontSize(this.pdfConfig.typo.large);
                pdf.text('ทดสอบการสร้าง pdf โดย jsPdf', pdf_width / 2, pdf_position_y, null, null, "center");
                pdf_position_y += this.pdfConfig.typo.large * this.pdfConfig.lineHeight.large;
                
                pdf.setFontSize(this.pdfConfig.typo.normal);
                pdf.text('ทดสอบการสร้าง pdf โดย jsPdf', pdf_width / 2, pdf_position_y, null, null, "center");
                pdf_position_y += this.pdfConfig.typo.normal * this.pdfConfig.lineHeight.normal;
                
                pdf.setFont('THSarabunNew','normal');
                pdf.setTextColor('#000000');
                pdf.setFontSize(this.pdfConfig.typo.normal);
                                
                const longText = '๒.๑ บก.ทท. (สส.ทหาร) จ้างพัฒนาระบบ ที่ Learning Management System และ Online Course ตามโครงการพัฒนาโครงสร้างพื้นฐานด้านเทคโนโลยีสารสนเทศ ประจำปีงบประมาณ พ.ศ. ๒๕๖๗ จำนวน ๑ งาน กับ บริษัท เพอร์เฟคคอมพิวเตอร์โซลูชัน จำกัด เป็นจำนวนเงินรวมทั้งสิ้น ๑๖,๑๔๘,๘๐๐.- บาท (สิบหกล้าน หนึ่งแสนสี่หมื่นแปดพันแปดร้อยบาทถ้วน) สัญญาจ้างฯ เลขที่ ๕๒/๒๕๖๗ ลง ๒๓ ส.ค. ๖๗ โดย สส.ทหาร รับไว้ใช้ในราชการ เมื่อ ๒ ก.ค. ๖๘ พร้อมการรับประกันความเสียหาย เป็นระยะเวลา ๓ ปี เริ่ม ๒ ก.ค. ๖๘ ครบกำหนด ๑ ก.ค. ๗๑ หากมีความชำรุดบกพร่องของพัสดุให้หน่วยผู้ใช้งานโดยตรงแจ้ง กจห.สส.ทหาร เพื่อประสาน บริษัท พอร์เฟคคอมพิวเตอร์โซลูชัน จำกัด ให้เร่งดำเนินการแก้ไขให้สามารถใช้ในราชการได้ตามที่สัญญาฯ กำหนด ก่อนสิ้นสุดระยะเวลารับประกัน ๓๐ วัน';                               
                const maxWidth = pdf_width - margin_l - margin_r;
                                const indent = 25; // mm

                                const paragraphText = (longText ?? '').toString();
                                if (paragraphText.trim()) {
                                    const fontSize = this.pdfConfig.typo.normal;
                                    const lineHeightFactor = this.pdfConfig.lineHeight.normal;
                                    const firstLineWidth = Math.max(1, maxWidth - indent);
                                    const firstLineSplit = pdf.splitTextToSize(paragraphText, firstLineWidth);
                                    const firstLine = firstLineSplit[0] || '';
                                    const scaleFactor = pdf.internal.scaleFactor || 1;
                                    const lineHeight = (fontSize * lineHeightFactor) / scaleFactor;

                                    pdf.text(firstLine, margin_l + indent, pdf_position_y, { maxWidth,
                                            align: 'justify',
                                            lineHeightFactor});

                                    const remainingText = paragraphText.slice(firstLine.length).trimStart();
                                    if (remainingText) {
                                        pdf.text(remainingText, margin_l, pdf_position_y + lineHeight, {
                                            maxWidth,
                                            align: 'justify',
                                            lineHeightFactor
                                        });

                                        const remainingLines = pdf.splitTextToSize(remainingText, maxWidth);
                                        pdf_position_y += lineHeight + (remainingLines.length * lineHeight);
                                    } else {
                                        pdf_position_y += lineHeight;
                                    }
                                }

                pdf.setTextColor('red');
                pdf.setFontSize(this.pdfConfig.typo.small);
                
                const longText2 = 'ทดสอบการสร้าง pdf โดย jsPdfddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddfdfdsf dfdsfsadf dsfsdfsadfasd dfsdfasdfsad fdsfsdfsdaf dddddddddddddddddddddddddddddd';
                const paragraphText2 = (longText2 ?? '').toString();
                if (paragraphText2.trim()) {
                    const fontSize = this.pdfConfig.typo.small;
                    const lineHeightFactor = this.pdfConfig.lineHeight.small;
                    const firstLineWidth = Math.max(1, maxWidth - indent);
                    const firstLineSplit = pdf.splitTextToSize(paragraphText2, firstLineWidth);
                    const firstLine = firstLineSplit[0] || '';
                    const scaleFactor = pdf.internal.scaleFactor || 1;
                    const lineHeight = (fontSize * lineHeightFactor) / scaleFactor;

                    pdf.text(firstLine, margin_l + indent, pdf_position_y, { maxWidth, align: 'justify', lineHeightFactor });

                    const remainingText = paragraphText2.slice(firstLine.length).trimStart();
                    if (remainingText) {
                        pdf.text(remainingText, margin_l, pdf_position_y + lineHeight, { maxWidth, align: 'justify', lineHeightFactor });
                        const remainingLines = pdf.splitTextToSize(remainingText, maxWidth);
                        pdf_position_y += lineHeight + (remainingLines.length * lineHeight);
                    } else {
                        pdf_position_y += lineHeight;
                    }
                }

                pdf.addPage();
                pdf_position_y = this.pdfConfig.margin.t;
                pdf.setFont('Sarabun','normal');
                pdf.setFontSize(this.pdfConfig.typo.header);
                pdf.setTextColor('#025955');
                pdf.text('ทดสอบการสร้าง pdf โดย jsPdf', pdf_width / 2, pdf_position_y, null, null, "center");

                setTimeout(() => {
                    pdf.setFont('Sarabun','normal');
                    pdf.setFontSize(this.pdfConfig.typo.small);
                    pdf.setTextColor('#000');
                    const textDate = (new Date()).toString();
                    
                    const pages = pdf.internal.getNumberOfPages();

                    for (let j = 1; j < pages + 1 ; j++) {
                        pdf.setPage(j);
                        pdf.text(`วันเวลา : ${textDate}`, margin_l, pdf_height - 10);
                        pdf.text(`หน้า ${j} จาก ${pages}`, pdf_width - margin_r, pdf_height - 10, null, null, "right");
                    }

                    pdf.save(Date.now() + ".pdf");
                }, 0);

            }
            catch (err) {
                console.log(err);
            }
        },
        async genarateHTML(){
            try {
                let pdf = new jsPDF(this.pdfOption);

                let pdfCanvas1 = this.$refs.pdfCanvas1;
                await html2canvas(pdfCanvas1, this.canvasOption).then((canvas) => {
                    pdf.addImage(canvas.toDataURL('image/jpeg', 1.0), 'JPEG', 0 , 0);
                });

                setTimeout(() => {
                    //download pdf file
                    pdf.save(Date.now() + ".pdf");
                },0);
            }
            catch (err) {
                console.log(err);
            }
        },
        async genarateImage(){
            try {
                await html2canvas(this.$refs.pdfCanvas1, this.canvasOption).then((canvas) => {
                    let imgCanvas = this.$refs.imgCanvas;
                    imgCanvas.src = canvas.toDataURL('image/jpeg', 1.0);
                });
            }
            catch (e) {
                console.log(e);
            }
        }
    }
}
</script>

<style>

</style>
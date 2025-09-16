https://www.jitbit.com/alexblog/249-now-thats-what-i-call-a-hacker/ -----
import React, { useState, useMemo } from "react";
import { motion, AnimatePresence } from "framer-motion";
import {
  SKIN_TYPES,
  CONCERNS,
  SUB_CONCERNS,
  LIFESTYLE,
  ROUTINE_LEVELS,
  GOALS,
} from "../../utils/SkinInsightData";
import RecommendedProductCard from "./RecommendedProductCard";
import Fadeloader from "@/components/common/Fadeloader";
import api from "@/utils/api";

const pageVariants = {
  initial: { opacity: 0, y: 12 },
  animate: { opacity: 1, y: 0 },
  exit: { opacity: 0, y: -12 },
  transition: { duration: 0.25 },
};

const OptionCard = ({ active, label, onClick }) => (
  <motion.button
    type="button"
    whileHover={{ scale: 1.02 }}
    whileTap={{ scale: 0.97 }}
    onClick={onClick}
    className={`w-full rounded-xl border p-4 text-left transition
      ${active ? "border-[#8C6C54] bg-[#8C6C54]/10" : "border-zinc-200 bg-white hover:border-zinc-300"}`}
  >
    {label}
  </motion.button>
);

const MultiPill = ({ checked, label, onChange }) => (
  <motion.label
    whileHover={{ scale: 1.05 }}
    className={`cursor-pointer inline-flex items-center gap-2 rounded-full border px-4 py-2 text-sm
      ${checked ? "bg-[#8C6C54]/10 border-[#8C6C54] text-[#8C6C54]" : "bg-white border-zinc-200 text-zinc-700"}`}
  >
    <input type="checkbox" checked={checked} onChange={onChange} className="hidden" />
    {label}
  </motion.label>
);

export default function SkinInsightQuiz() {
  const [step, setStep] = useState(0);
  const [answers, setAnswers] = useState({
    ageGroup: "",
    email: "",
    skinType: "",
    concerns: [],
    subConcerns: [],
    lifestyle: [],
    routineLevel: "",
    goal: [],
  });
  const [errors, setErrors] = useState({});
  const [showResults, setShowResults] = useState(false);
  const [loading, setLoading] = useState(false);
  const [recommendations, setRecommendations] = useState([]);
  const [selectedProducts, setSelectedProducts] = useState([]);

  const steps = [
    {
      key: "ageEmail",
      title: "Tell us about you",
      render: () => (
        <>
          <div>
            <h3>How old are you?</h3>
            {["18-29", "30-39", "40-49", "50+"].map((opt) => (
              <OptionCard
                key={opt}
                active={answers.ageGroup === opt}
                label={opt}
                onClick={() => setAnswers((a) => ({ ...a, ageGroup: opt }))}
              />
            ))}
          </div>
          <div>
            <h3>Your Email</h3>
            <input
              value={answers.email}
              onChange={(e) => setAnswers((a) => ({ ...a, email: e.target.value }))}
              className="w-full border px-4 py-2 rounded-lg"
              placeholder="Enter your email"
            />
          </div>
        </>
      ),
      validate: () => {
        const errs = {};
        if (!answers.ageGroup) errs.ageGroup = "Pick your age";
        if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(answers.email)) errs.email = "Enter valid email";
        return errs;
      },
    },
    {
      key: "skinType",
      title: "What best describes your skin type?",
      options: SKIN_TYPES,
      type: "single",
    },
    {
      key: "concerns",
      title: "What are your main skincare concerns?",
      options: CONCERNS,
      type: "multi",
    },
    {
      key: "subConcerns",
      title: "Let’s go deeper — specific issues?",
      render: () =>
        answers.concerns.map((cid) => (
          <div key={cid}>
            <h4>{CONCERNS.find((c) => c.id === cid)?.label}</h4>
            {SUB_CONCERNS[cid]?.map((sub) => (
              <MultiPill
                key={sub}
                checked={answers.subConcerns.includes(sub)}
                label={sub}
                onChange={() =>
                  setAnswers((a) => ({
                    ...a,
                    subConcerns: a.subConcerns.includes(sub)
                      ? a.subConcerns.filter((x) => x !== sub)
                      : [...a.subConcerns, sub],
                  }))
                }
              />
            ))}
          </div>
        )),
    },
    { key: "lifestyle", title: "Which apply to your daily life?", options: LIFESTYLE, type: "multi" },
    { key: "routineLevel", title: "How’s your skincare routine?", options: ROUTINE_LEVELS, type: "single" },
    { key: "goal", title: "Your top skincare goal?", options: GOALS, type: "multi" },
  ];

  const handleNext = async () => {
    const current = steps[step];
    const errs = current.validate ? current.validate() : {};
    setErrors(errs);
    if (Object.keys(errs).length) return;
    if (step < steps.length - 1) setStep((s) => s + 1);
    else {
      setShowResults(true);
      await fetchRecommendations();
    }
  };

  const fetchRecommendations = async () => {
    try {
      setLoading(true);
      const res = await api.post("/recommendations/skin-form", {
        ...answers,
        skinTypes: [answers.skinType],
        routine: [answers.routineLevel],
      });
      const recs = res.data?.recommendations || [];
      setRecommendations(recs);
      setSelectedProducts(recs.map((p) => p.id || p._id));
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="max-w-3xl mx-auto p-6">
      {!showResults ? (
        <AnimatePresence mode="wait">
          <motion.div key={step} {...pageVariants} className="space-y-4">
            <h2 className="text-xl font-semibold">{steps[step].title}</h2>
            {steps[step].render ? (
              steps[step].render()
            ) : steps[step].type === "single" ? (
              steps[step].options.map((opt) => (
                <OptionCard
                  key={opt.id || opt}
                  active={answers[steps[step].key] === (opt.id || opt)}
                  label={opt.label || opt}
                  onClick={() => setAnswers((a) => ({ ...a, [steps[step].key]: opt.id || opt }))}
                />
              ))
            ) : (
              steps[step].options.map((opt) => {
                const checked = answers[steps[step].key].includes(opt.id);
                return (
                  <MultiPill
                    key={opt.id}
                    checked={checked}
                    label={opt.label}
                    onChange={() =>
                      setAnswers((a) => ({
                        ...a,
                        [steps[step].key]: checked
                          ? a[steps[step].key].filter((x) => x !== opt.id)
                          : [...a[steps[step].key], opt.id],
                      }))
                    }
                  />
                );
              })
            )}
            {errors[steps[step].key] && <p className="text-red-500">{errors[steps[step].key]}</p>}
            <div className="flex justify-between mt-6">
              <button disabled={step === 0} onClick={() => setStep((s) => s - 1)}>Back</button>
              <button onClick={handleNext}>{step < steps.length - 1 ? "Next" : "See Recommendations"}</button>
            </div>
          </motion.div>
        </AnimatePresence>
      ) : (
        <div>
          <h2>Your Personalized Products</h2>
          {loading ? (
            <Fadeloader />
          ) : (
            <div className="grid gap-4 md:grid-cols-3">
              {recommendations.map((p) => (
                <RecommendedProductCard key={p.id || p._id} product={p} selected={selectedProducts.includes(p.id)} />
              ))}
            </div>
          )}
        </div>
      )}
    </div>
  );
}
-----------------------------------------------------------
https://buildabear.com.au/
https://in.pinterest.com/pin/3588874695957911/
=========================================
https://modkids.casethemes.net/home-02/
https://modkids.casethemes.net/
-------------------code for rope and cards need to modify--

import React from "react";

const programs = [
  {
    category: "Sport Program",
    title: "Grow your own wings",
    image: "https://i.ibb.co/vjtcP4n/kids1.jpg",
    bg: "bg-blue-200",
  },
  {
    category: "Kids Play",
    title: "Education Program",
    image: "https://i.ibb.co/jr3XTr6/kids2.jpg",
    bg: "bg-yellow-200",
  },
  {
    category: "Baby Essentials",
    title: "Life Skills for kids",
    image: "https://i.ibb.co/6Hj4vQ6/kids3.jpg",
    bg: "bg-orange-200",
  },
  {
    category: "Kids Skill",
    title: "Nurturing Caregiving",
    image: "https://i.ibb.co/6rZn8dM/kids4.jpg",
    bg: "bg-green-200",
  },
  {
    category: "Kids Playing",
    title: "Enriching Learning",
    image: "https://i.ibb.co/0GnydLJ/kids5.jpg",
    bg: "bg-pink-200",
  },
];

export default function EducationalPrograms() {
  return (
    <div className="relative w-full py-12 bg-gray-50">
      {/* Rope (SVG curve) */}
      <svg
        className="absolute top-0 left-0 w-full h-40"
        viewBox="0 0 1000 200"
        preserveAspectRatio="none"
      >
        <path
          d="M0,100 C250,200 750,0 1000,100"
          stroke="#6b7280"
          strokeWidth="3"
          fill="transparent"
        />
      </svg>

      <div className="relative flex justify-center gap-6 mt-10 flex-wrap">
        {programs.map((program, idx) => (
          <div
            key={idx}
            className={`${program.bg} w-56 rounded-2xl shadow-lg transform rotate-${idx % 2 === 0 ? "2" : "-2"} relative`}
          >
            {/* Clip */}
            <div className="absolute -top-6 left-1/2 -translate-x-1/2 w-6 h-6 bg-purple-700 rounded-t-md" />

            {/* Content */}
            <img
              src={program.image}
              alt={program.title}
              className="w-full h-36 object-cover rounded-t-2xl"
            />
            <div className="p-3">
              <p className="text-sm text-red-500 font-medium">{program.category}</p>
              <h3 className="text-lg font-bold text-gray-800">
                {program.title}
              </h3>
            </div>
          </div>
        ))}
      </div>
    </div>
  );
}

----------------------------------------------------------------
smart ingridients ,what makes us differennt form others
https://webflow.com/templates/html/gummie-website-template
----------------
https://in.pinterest.com/pin/4081455907100135/
https://in.pinterest.com/pin/4433299629914231/ home page
https://in.pinterest.com/pin/11047961581415686/
https://in.pinterest.com/pin/3729612268863251/ single page
https://in.pinterest.com/pin/914862421035307/ for about us page
https://in.pinterest.com/pin/59743132553776153/
https://in.pinterest.com/pin/1618549864251561/
https://in.pinterest.com/pin/2744449769305768/ cards
0-------------------------------------------------------
product card---update newSubscriptioncar----
------------------------

import React, { useEffect, useRef, useState } from "react";
import { Swiper, SwiperSlide } from "swiper/react";
import { Thumbs } from "swiper/modules";
import { ArrowLeft, ArrowRight } from "lucide-react";
import img1 from "../../assets/imgs/bg1.webp";
import img2 from "../../assets/imgs/bg2.webp";

import "swiper/css";
import "swiper/css/thumbs";

const dummyImages = [
  { url: img1, alt: "Product 1" },
  { url: img1, alt: "Product 2" },
  { url: img2, alt: "Product 3" },
  { url: img2, alt: "Product 4" },
];

export default function ProductCard() {
  const [thumbsV, setThumbsV] = useState(null); // vertical thumbs (desktop)
  const [thumbsH, setThumbsH] = useState(null); // horizontal thumbs (mobile)
  const [isDesktop, setIsDesktop] = useState(
    typeof window !== "undefined" ? window.innerWidth >= 768 : true
  );

  const [isBeginning, setIsBeginning] = useState(true);
  const [isEnd, setIsEnd] = useState(false);

  const mainSwiperRef = useRef(null);

  // media query to toggle desktop/mobile
  useEffect(() => {
    if (typeof window === "undefined") return;
    const mq = window.matchMedia("(min-width: 768px)");
    const handler = (e) => setIsDesktop(e.matches);
    handler(mq); // initialize
    if (mq.addEventListener) {
      mq.addEventListener("change", handler);
      return () => mq.removeEventListener("change", handler);
    } else {
      mq.addListener(handler);
      return () => mq.removeListener(handler);
    }
  }, []);

  // pick the correct thumbs swiper instance for the main swiper
  const currentThumbs = isDesktop ? thumbsV : thumbsH;

  return (
    <div className="flex flex-col md:flex-row gap-6 w-full p-6 md:p-12 bg-gray-50 rounded-xl">
      {/* LEFT block: thumbs + main slider */}
      <div className="w-full md:w-1/2 flex flex-col md:flex-row items-start gap-4 ">
        {/* Vertical thumbs - only render on desktop */}
        {isDesktop && (
          <div className="hidden md:block w-20">
            <Swiper
              direction="vertical"
              spaceBetween={10}
              slidesPerView={4}
              watchSlidesProgress={true}
              modules={[Thumbs]}
              onSwiper={(s) => setThumbsV(s)}
              className="w-20 h-[50vh] md:h-[60vh]"
            >
              {dummyImages.map((img, idx) => (
                <SwiperSlide key={idx}>
                  <button
                    type="button"
                    onClick={() => mainSwiperRef.current?.slideTo(idx)}
                    className="w-full h-20 p-1"
                  >
                    <img
                      src={img.url}
                      alt={`thumb-${idx}`}
                      className=" w-full h-20  object-contain rounded-md border border-gray-200"
                    />
                  </button>
                </SwiperSlide>
              ))}
            </Swiper>
          </div>
        )}

        {/* Main slider */}
        <div className="relative  md:flex-1 md:min-w-0 w-full  ">
          <Swiper
            onSwiper={(s) => {
              mainSwiperRef.current = s;
              setIsBeginning(s.isBeginning);
              setIsEnd(s.isEnd);
            }}
            onSlideChange={(s) => {
              setIsBeginning(s.isBeginning);
              setIsEnd(s.isEnd);
            }}
            spaceBetween={20}
            slidesPerView={1}
            thumbs={currentThumbs ? { swiper: currentThumbs } : undefined}
            modules={[Thumbs]}
            className="rounded-2xl"
          >
            {dummyImages.map((img, idx) => (
              <SwiperSlide key={idx} className="">
                <div className="w-full h-[50vh] md:h-[60vh] bg-gray-100 rounded-2xl flex items-center justify-center overflow-hidden">
                  <img src={img.url} alt={img.alt} className="object-contain w-full h-full" />
                </div>
              </SwiperSlide>
            ))}
          </Swiper>

            {/* arrows - disabled visually & functionally at ends */}
          <div className="absolute inset-y-0 left-0 right-0 z-10 flex justify-between items-center px-4 pointer-events-none">
            <button
              type="button"
              onClick={() => !isBeginning && mainSwiperRef.current?.slidePrev()}
              className={`pointer-events-auto bg-white/90 rounded-full p-1 shadow transition transform ${
                isBeginning ? "opacity-40 cursor-not-allowed" : "hover:scale-110"
              }`}
              aria-disabled={isBeginning}
            >
              <ArrowLeft />
            </button>

            <button
              type="button"
              onClick={() => !isEnd && mainSwiperRef.current?.slideNext()}
              className={`pointer-events-auto bg-white/90 rounded-full p-1 shadow transition transform ${
                isEnd ? "opacity-40 cursor-not-allowed" : "hover:scale-110"
              }`}
              aria-disabled={isEnd}
            >
              <ArrowRight />
            </button>
          </div>
        </div>

        {/* Horizontal thumbs - only render on mobile (below main) */}
        {!isDesktop && (
          <div className="w-full block md:hidden mt-4">
            <Swiper
              direction="horizontal"
              spaceBetween={10}
              slidesPerView={"auto"}
              watchSlidesProgress={true}
              modules={[Thumbs]}
              onSwiper={(s) => setThumbsH(s)}
              className="w-full"
            >
              {dummyImages.map((img, idx) => (
                <SwiperSlide key={idx} style={{ width: 100 }}>
                  <button
                    type="button"
                    onClick={() => mainSwiperRef.current?.slideTo(idx)}
                    className="w-full h-24 p-1"
                  >
                    <img
                      src={img.url}
                      alt={`thumb-mobile-${idx}`}
                      className="w-full h-full object-contain rounded-md border border-gray-200"
                    />
                  </button>
                </SwiperSlide>
              ))}
            </Swiper>
          </div>
        )}
      </div>

      {/* RIGHT: product details (keep yours here) */}
      <div className="w-full md:w-1/2 flex flex-col gap-4">
        <h2 className="text-2xl font-bold">One-time Purchase</h2>
        <p className="text-gray-600">₹849.00 / per bottle</p>
        <button className="bg-black text-white py-3 rounded-lg hover:scale-105 transition">Add To Cart</button>
      </div>
    </div>
  );
}
----------------------------------------------------
NewTestimonial ----------------
import React from "react";
import { Swiper, SwiperSlide } from "swiper/react";
import { Navigation } from "swiper/modules";
import "swiper/css";
import "swiper/css/navigation";
import "./testimonialSwiper.css";
import { ArrowLeft, ArrowRight } from "lucide-react";
import { motion as m } from "framer-motion";
import { FaQuoteLeft, FaQuoteRight } from "react-icons/fa6";
import { fadeInUp } from "../../components/common/animation";
const testimonials = [
  {
    id: 1,
    name: "Sophia R.",
    description:
      "My 8-year-old daughter has always struggled with her eyesight, but after using Eyebeam for just 90 days, I noticed a huge improvement! She doesn’t squint as much anymore, and her eye health has visibly improved. ",
    image:
      "https://img.freepik.com/premium-photo/portrait-happy-indian-asian-young-family-while-standing-against-wall-parents-with-two-daughters-looking-camera_466689-7316.jpg?w=740",
    rating: 5,
  },
  {
    id: 2,
    name: "James M.",
    description:
      "As a mom, I always worry about my child's eye health, especially with so much screen time. Eyebeam gummies have been a game changer! My son loves the cranberry flavor, and I can see his vision getting sharper every day. Highly recommend!",
    image:
      "https://img.freepik.com/premium-photo/man-with-his-arms-crossed-word-word-front_916191-428856.jpg?w=740",
    rating: 4.5,
  },
  {
    id: 3,
    name: "Emily T.",
    description:
      "I’ve been looking for a product that supports my son's bone health and helps him grow stronger, and Bonevito is exactly what he needed. The gummies are fun, delicious, and packed with all the nutrients he needs. I’m so glad we found them",
    image:
      "https://img.freepik.com/premium-photo/family-three-smiling-mother-father-son_1029679-167596.jpg?w=740",
    rating: 5,
  },
  {
    id: 4,
    name: "Michael B.",
    description:
      "My 6-year-old loves playing soccer, but I noticed her bones weren’t as strong as they should be. Since we started giving her Bonevito gummies, I’ve seen a huge improvement in her energy levels and strength. Plus, she loves the mango flavor!",
    image:
      "https://img.freepik.com/premium-photo/family-poses-photo-with-girl_911060-135370.jpg?w=740",
    rating: 4.8,
  },
  {
    id: 5,
    name: "Olivia K.",
    description:
      "As a mom with a picky eater, I’m always looking for ways to ensure my child gets all the nutrients they need. Vitameez gummies have been perfect! They boost his immunity and promoted growth, all while tasting amazing. He asks for them every day!",
    image:
      "https://img.freepik.com/premium-photo/family-portrait-with-two-people-posing-photo_1290348-2157.jpg?w=740",
    rating: 5,
  },
  {
    id: 6,
    name: "Sophia R.",
    description:
      "My son is always on the go, and I was worried he wasn’t getting enough nutrition. Vitameez has been an absolute helping hand. It’s packed with vitamins, and he loves the strawberry flavor. I’m confident he’s getting everything he needs.",
    image:
      "https://img.freepik.com/premium-photo/portrait-happy-indian-asian-young-family-while-standing-against-wall-parents-with-two-daughters-looking-camera_466689-7316.jpg?w=740",
    rating: 5,
  },
  {
    id: 7,
    name: "James M.",
    description:
      "I was looking for something natural to help my child fight with infections, and Imunoprash gummies fit the bill perfectly! They love the taste, and I’m confident their immune systems are getting stronger every day. I highly recommend these!",
    image:
      "https://img.freepik.com/premium-photo/happy-father-spends-time-with-children-christmas-vacation_105751-13821.jpg?w=360",
    rating: 4.5,
  },
];
const TestimonialSwiper = () => {
  return (
    <div>
        <m.h2
                variants={fadeInUp}
                initial="hidden"
                whileInView="visible"
                className="text-3xl sm:text-4xl mb-10 md:text-5xl font-semibold text-woobyColor text-center pt-20 capitalize"
              >
                What Parents Say About Us?
              </m.h2>
    <div className="bg-[#f7f6f6c5] text-white py-10">
      <div className="max-w-3xl mx-auto px-4 relative">
        <Swiper
      modules={[Navigation]}
      loop={true}
      centeredSlides={true}
      slidesPerView={3}
      spaceBetween={0}
      navigation={{
        nextEl: ".testimonial-next",
        prevEl: ".testimonial-prev",
      }}
      className="outer-swiper"
        >
          {testimonials.map((item) => (
            <SwiperSlide key={item.id} className="testimonial-slide">
              <div className="bg-white text-black rounded-2xl shadow-lg overflow-hidden w-[300px]">
                <div className="">
                  <img
                    src={item.image}
                    alt={item.product}
                    className="w-full h-52 object-cover object-top"
                  />
                </div>
                {/* <div className="bg-gradient-to-r from-woobyOrange via-woobyOrangeLight to-woobyOrangeDark absolute backdrop:blur-2xl bottom-0 left-0 right-0  mx-2 mb-2 backdrop-blur-sm rounded-xl p-2 flex items-center gap-2 shadow-md"
                >
                  <img
                    src={item.image}
                    alt={item.image || item.product}
                    className="w-12 h-12 object-contain "
                  />

                  <div className="flex flex-col gap-2 w-full">
                    <p className="text-xs font-semibold text-gray-800 mr-5">
                     Gummy Bears
                      (Strawberry)
                    </p>

                    <div className="flex items-center justify-between gap-4 rounded-lg ">
                      <span className="text-green-600 font-bold text-lg ">
                      ₹229
                      </span>

                      <button
                        className="border-2 py-1 px-2 text-xs font-medium transition-colors rounded-lg cursor-pointer border-woobyColor text-woobyColor hover:bg-woobyColor hover:text-white"
                      >
                        Add to Cart
                      </button>
                    </div>
                  </div>
                </div> */}
                <div className="p-4">
                  <p className="font-semibold text-sm text-woobyColor">
                    {item.name}
                  </p>
                  <p className="font-bold text-base">{item.product}</p>
                  <p className="text-sm mt-2  text-gray-700">
                    {" "}
                    <FaQuoteLeft className="inline-block text-xl mr-2 text-woobyColor" />
                    {item.description}
                    <FaQuoteRight className="inline-block ml-2 text-woobyColor" />
                  </p>
                  <div className="flex justify-between items-center">
                    <div>
                      {/* <span className="text-lg font-bold">{item.price}</span> */}
                      <span className="text-sm text-gray-500 line-through ml-2">
                        {/* {item.oldPrice} */}
                      </span>
                    </div>
                    {/* <button className="bg-black text-white px-4 py-2 rounded-md text-sm">
                      Add to Cart
                    </button> */}
                  </div>
                </div>
              </div>
            </SwiperSlide>
          ))}
        </Swiper>
        <button className="testimonial-prev absolute -left-12 top-1/2 -translate-y-1/2 bg-white text-woobyColor shadow-lg rounded-full p-3 z-10">
        <ArrowLeft />
  </button>

  <button className="testimonial-next absolute -right-12 top-1/2 -translate-y-1/2 bg-white text-woobyColor shadow-lg rounded-full p-3 z-10">
  <ArrowRight />
  </button>
      </div>
    </div>

    </div>
  );
};

export default TestimonialSwiper;
---------------------------

----------------------------------------------------
product card ---------------------------
//one with filling color-----------------
// import React from "react";
// import { motion } from "framer-motion";
// import {
//   fadeInUp,
//   fadeInLeft,
//   fadeInRight,
//   scaleUp,
//   inViewOptions,
// } from "../../components/common/animation";
// import { PiTrademark } from "react-icons/pi";
// import { useEffect, useRef } from "react";
// import { Check } from 'lucide-react';



// const transition = {
//   duration: 4,
//   repeat: Infinity,
//   repeatType: "reverse",
//   ease: "easeInOut",
// };
// const ProductHighlight = ({ benefits }) => {
//   console.log("benefits", benefits);
//   const circleRef = useRef(null);

//   useEffect(() => {
//     const circle = circleRef.current;
//     if (circle) {
//       circle.classList.add("animate-spin-slow");
//     }
//   }, []);
//   const colorMap = {
//     red: "bg-[#9f2923] text-white border-[#9f2923]",
//     "#FF7821": "bg-[#FF7821] text-white border-[#FF7821]",
//     indigo: "bg-[#4b277d] text-white border-[#4b277d]",
//     green: "bg-[#1D8821] text-white border-[#1D8821]",
//   };
//   return (

//     <div className="flex items-center justify-center min-h-screen bg-white">
//       <div
//         className={`relative w-[400px] h-[400px] flex items-center justify-center  rounded-full border-${
//           colorMap[benefits?.bgColor]
//         }`}
//       >
//          <div className="absolute inset-0">
//           <svg xmlns="http://www.w3.org/2000/svg" width="400" height="400">
//           <motion.path
//   d="M 200 0 
//      C 55 10 0 130 5 200 
//      C 5 220 5 390 220 392 
//      C 300 380 400 300 390 200 
//      C 400 120 325 40 260 15"
//   fill="#f9d3ae"
//   strokeWidth="8"
//   stroke="#F7AB5E"
//   strokeLinecap="round"
//   initial={{ pathLength: 0 }}
//   animate={{ pathLength: 1 }}
//   transition={transition}
// />
//           </svg>
//           <motion.div
//             style={box}
//             initial={{ offsetDistance: "0%", scale: 1.2 }}
//             animate={{ offsetDistance: "100%", scale: 1 }}
//             transition={transition}
//           />
//         </div>


//         <div className=" max-lg:hidden  w-[400px] h-[400px] rounded-full absolute mx-auto innerOrbit-wrapper">
//           <div className="absolute right-86 top-1/4 -translate-y-1/2 innerProfile-wrapper  ">
//             <div className="flex justify-between items-center w-96 border-black gap-4">
//               <div className="flex flex-col justify-end items-end">
//                 <p className="text-[#F27635] font-bold">{benefits.leftBenefits[0].title}ww</p>
//                 <p className="text-[#F27635]">
//                   {benefits.leftBenefits[0].description}
//                 </p>
//               </div>
//               <div className="h-6 w-9 text-black bg-[#F27635] rounded-full animate ease-out"> <Check className="text-white p-1 "/></div>
//             </div>
//           </div>
//           {/* <div className="absolute -right-9 top-1/2 -translate-y-1/2 innerProfile-wrapper">
//                     <div className="flex justify-center items-center"><div className="h-5 w-5 text-black bg-amber-950 rounded-full"></div><p className="text-black">eee</p></div>

//         </div> */}
//           <div className="absolute  left-86 top-1/4 -translate-y-1/2 innerProfile-wrapper">
//             <div className="flex justify-between items-center  w-96 border-black gap-4">
//               <div className="h-6 w-9 text-black bg-[#F27635] rounded-full">   <Check className="text-white p-1"/></div>
//               <div className="border-black ">
//                 <p className="text-[#F27635] font-bold">{benefits.leftBenefits[1].title}ee</p>
//                 <p className="text-[#F27635]">
//                   {benefits.leftBenefits[1].description}
//                 </p>
//               </div>
//             </div>
//           </div>
//           <div className="absolute  left-83 top-[320px] -translate-y-1/2 innerProfile-wrapper">
//             <div className="flex justify-between items-center  w-96 gap-4">
//               <div className="h-6 w-8 text-black bg-[#F27635] rounded-full"> <Check className="text-white p-1"/></div>
//               <div >
//                 <p className="text-[#F27635] font-bold">{benefits.leftBenefits[2].title}oo</p>
//                 <p className="text-[#F27635]">
//                   {benefits.leftBenefits[2].description}
//                 </p>
//               </div>
//             </div>
//           </div>
//           <div className="absolute right-83 top-[320px] -translate-y-1/2 innerProfile-wrapper">
//             <div className="flex justify-between items-center  w-96 gap-4">
//               <div className="flex flex-col justify-end items-end">
//                 <p className="text-[#F27635] font-bold">{benefits.leftBenefits[3].title}</p>
//                 <p className="text-[#F27635]">
//                   {benefits.leftBenefits[3].description}
//                 </p>
//               </div>
//               <div className="h-6 w-8 text-black bg-[#F27635] rounded-full"> <Check className="text-white p-1 "/></div>
//             </div>
//           </div>
//           {/* <div className="absolute right-94 top-1/2 -translate-y-1/2 innerProfile-wrapper">
//                     <div className="flex justify-center items-center"><p className="text-black">kkk</p><div className="h-5 w-8 text-black bg-[#F27635] rounded-full"></div></div>
          
//         </div> */}
//         </div>

//         <img
//           src={benefits.benifitImg}
//           alt="NutriBears Bottle"
//           className="w-full h-full object-contain drop-shadow-lg rounded-full"
//         />
//       </div>
//     </div>
//   );
// };
// const box = {
//   width: 20,
//   height: 20,
//   backgroundColor: "#8df0cc",
//   borderRadius: "50%",
//   position: "absolute",
//   top: 0,
//   left: 0,
//   offsetPath: `path("M 200 20 C 100 20 40 100 40 200 C 40 300 120 380 200 380 C 280 380 360 300 360 200 C 360 100 300 60 200 60")`,
// };

// export default ProductHighlight;



// ----------without circle-just border
// import React from "react";
// import { motion } from "framer-motion";
// import {
//   fadeInUp,
//   fadeInLeft,
//   fadeInRight,
//   scaleUp,
//   inViewOptions,
// } from "../../components/common/animation";
// import { PiTrademark } from "react-icons/pi";
// import { useEffect, useRef } from "react";
// import { Check } from 'lucide-react';



// const transition = {
//   duration: 4,
//   repeat: Infinity,
//   repeatType: "reverse",
//   ease: "easeInOut",
// };
// const ProductHighlight = ({ benefits }) => {
//   console.log("benefits", benefits);
//   const circleRef = useRef(null);

//   useEffect(() => {
//     const circle = circleRef.current;
//     if (circle) {
//       circle.classList.add("animate-spin-slow");
//     }
//   }, []);
//   const colorMap = {
//     red: "bg-[#9f2923] text-white border-[#9f2923]",
//     "#FF7821": "bg-[#FF7821] text-white border-[#FF7821]",
//     indigo: "bg-[#4b277d] text-white border-[#4b277d]",
//     green: "bg-[#1D8821] text-white border-[#1D8821]",
//   };
//   return (

//     <div className="flex items-center justify-center min-h-screen bg-white">
//       <div
//         className={`relative w-[400px] h-[400px] flex items-center justify-center border-r-4 border-l-4 rounded-full border-${
//           colorMap[benefits?.bgColor]
//         }`}
//       >
        


//         <div className=" max-lg:hidden border-1  w-[400px] h-[400px] rounded-full absolute mx-auto innerOrbit-wrapper">
//           <div className="absolute right-86 top-1/4 -translate-y-1/2 innerProfile-wrapper  ">
//             <div className="flex justify-between items-center w-96 border-black gap-4">
//               <div className="flex flex-col justify-end items-end">
//                 <p className="text-black">{benefits.leftBenefits[0].title}ww</p>
//                 <p className="text-black">
//                   {benefits.leftBenefits[0].description}
//                 </p>
//               </div>
//               <div className="h-5 w-9 text-black bg-[#F27635] rounded-full animate ease-out"> <Check className="text-white p-1 "/></div>
//             </div>
//           </div>
//           {/* <div className="absolute -right-9 top-1/2 -translate-y-1/2 innerProfile-wrapper">
//                     <div className="flex justify-center items-center"><div className="h-5 w-5 text-black bg-[#F27635] rounded-full"></div><p className="text-black">eee</p></div>

//         </div> */}
//           <div className="absolute  left-86 top-1/4 -translate-y-1/2 innerProfile-wrapper">
//             <div className="flex justify-between items-center  w-96 border-black gap-4">
//               <div className="h-5 w-9 text-black bg-amber-950 rounded-full">   <Check className="text-white p-1"/></div>
//               <div className="border-black ">
//                 <p className="text-black">{benefits.leftBenefits[1].title}ee</p>
//                 <p className="text-black">
//                   {benefits.leftBenefits[1].description}
//                 </p>
//               </div>
//             </div>
//           </div>
//           <div className="absolute  left-83 top-[320px] -translate-y-1/2 innerProfile-wrapper">
//             <div className="flex justify-between items-center  w-96 gap-4">
//               <div className="h-5 w-8 text-black bg-amber-950 rounded-full"> <Check className="text-white p-1"/></div>
//               <div >
//                 <p className="text-black">{benefits.leftBenefits[2].title}oo</p>
//                 <p className="text-black">
//                   {benefits.leftBenefits[2].description}
//                 </p>
//               </div>
//             </div>
//           </div>
//           <div className="absolute right-83 top-[320px] -translate-y-1/2 innerProfile-wrapper">
//             <div className="flex justify-between items-center  w-96 gap-4">
//               <div className="flex flex-col justify-end items-end">
//                 <p className="text-black">{benefits.leftBenefits[3].title}</p>
//                 <p className="text-black">
//                   {benefits.leftBenefits[3].description}
//                 </p>
//               </div>
//               <div className="h-5 w-8 text-black bg-amber-950 rounded-full"> <Check className="text-white p-1 "/></div>
//             </div>
//           </div>
//           {/* <div className="absolute right-94 top-1/2 -translate-y-1/2 innerProfile-wrapper">
//                     <div className="flex justify-center items-center"><p className="text-black">kkk</p><div className="h-5 w-8 text-black bg-amber-950 rounded-full"></div></div>
          
//         </div> */}
//         </div>

//         <img
//           src={benefits.benifitImg}
//           alt="NutriBears Bottle"
//           className="w-full h-full object-contain drop-shadow-lg rounded-full"
//         />
//       </div>
//     </div>
//   );
// };


// export default ProductHighlight;
// --------------------------------------------

// one with the round circles
import React from "react";
import { motion } from "framer-motion";

const bottleImg = "/photo_2025-09-16_23-15-16-removebg-preview.png";

export default function ProductHighlight({benefits}) {
    const size = 600;
    const outerRadius = 250;
    const midRadius = 200;
    const innerRadius = 110;

    const features = [
        { title: "Natural Sugar", subtitle: "of just 1.5g/gummy" },
        { title: "100%", subtitle: "Vegetarian" },
        { title: "No", subtitle: "Preservatives" },
        { title: "Pectin", subtitle: "based" },
        { title: "Natural", subtitle: "colours & flavours" },
        { title: "Essential", subtitle: "vitamins & minerals" },
    ];

    const leftFeatures = features.slice(0, 3);
    const rightFeatures = features.slice(3);

    function tickPos(index, total, side) {
        const angleStep = Math.PI / (total + 1);
        const angle = angleStep * (index + 1) + (side === "left" ? Math.PI / 2 : -Math.PI / 2);
        const tickX = size / 2 + outerRadius * Math.cos(angle);
        const tickY = size / 2 + outerRadius * Math.sin(angle);
        return { tickX, tickY };
    }

    // Helper to generate crystals
    const generateCrystals = (count, radius) =>
        Array.from({ length: count }).map((_, i) => {
            const angle = (i / count) * 2 * Math.PI;
            return {
                x: radius * Math.cos(angle),
                y: radius * Math.sin(angle),
            };
        });

    const innerCrystals = generateCrystals(6, innerRadius - 10);
    const midCrystals = generateCrystals(8, midRadius - 10);
    const outerCrystals = generateCrystals(10, outerRadius - 10);

    return (
        <div className="min-h-screen bg-white flex items-center justify-center p-6">
            <div className="relative" style={{ width: size, height: size }}>
                {/* INNER CIRCLE - Fast */}
                <motion.div
                    className="absolute rounded-full"
                    style={{
                        width: innerRadius * 2,
                        height: innerRadius * 2,
                        left: size / 2 - innerRadius,
                        top: size / 2 - innerRadius,
                        background: "radial-gradient(circle, rgba(255, 180, 120, 0.12) 0%, rgba(255, 122, 24, 0.05) 80%)",
                        border: "1px solid rgba(255, 122, 24, 0.3)",
                        backdropFilter: "blur(4px)",
                    }}
                    animate={{ rotate: -360 }}
                    transition={{ duration: 12, repeat: Infinity, ease: "linear" }}
                >
                    {innerCrystals.map((c, i) => (
                        <div
                            key={i}
                            className="absolute w-2 h-2 rounded-full bg-orange-400 shadow-md"
                            style={{
                                left: "50%",
                                top: "50%",
                                transform: `translate(${c.x}px, ${c.y}px)`,
                            }}
                        />
                    ))}
                </motion.div>
{/* MIDDLE CIRCLE - Medium */}
                <motion.div
                    className="absolute rounded-full"
                    style={{
                        width: midRadius * 2,
                        height: midRadius * 2,
                        left: size / 2 - midRadius,
                        top: size / 2 - midRadius,
                        background: "radial-gradient(circle, rgba(255, 90, 90, 0.10) 0%, rgba(255, 60, 60, 0.04) 80%)",
                        border: "1px solid rgba(255, 60, 60, 0.3)",
                        backdropFilter: "blur(3px)",
                    }}
                    animate={{ rotate: 360 }}
                    transition={{ duration: 20, repeat: Infinity, ease: "linear" }}
                >
                    {midCrystals.map((c, i) => (
                        <div
                            key={i}
                            className="absolute w-2 h-2 rounded-full bg-pink-400 shadow"
                            style={{
                                left: "50%",
                                top: "50%",
                                transform: `translate(${c.x}px, ${c.y}px)`,
                            }}
                        />
                    ))}
                </motion.div>

                {/* OUTER CIRCLE - Slowest */}
                <motion.div
                    className="absolute rounded-full"
                    style={{
                        width: outerRadius * 2,
                        height: outerRadius * 2,
                        left: size / 2 - outerRadius,
                        top: size / 2 - outerRadius,
                        borderWidth: 3,
                        borderStyle: "solid",
                        borderRadius: "50%",
                        background: "radial-gradient(circle, rgba(255, 255, 255, 0.02), rgba(255, 200, 150, 0.03))",
                        backdropFilter: "blur(2px)",
                    }}
                    animate={{
                        rotate: -360,
                        borderColor: ["#ff7a18", "#ff0000ff", "#ffffffff", "#ff7a18"],
                    }}
                    transition={{ duration: 28, repeat: Infinity, ease: "linear" }}
                >
                    
                </motion.div>

                {/* LEFT FEATURES */}
                {leftFeatures.map((f, i) => {
                    const { tickX, tickY } = tickPos(i, leftFeatures.length, "left");
                    return (
                        <div key={i} className="absolute">
                            <div
                                style={{
                                    left: tickX,
                                    top: tickY,
                                    transform: "translate(-50%, -50%)",
                                    position: "absolute",
                                }}
                                className="w-3 h-3 rounded-full bg-orange-500 border border-white"
                            ></div>
                            <div
                                style={{
                                    left: tickX - 50,
                                    top: tickY,
                                    transform: "translate(-100%, -50%)",
                                    position: "absolute",
                                }}
                                className="text-right"
                            >
                                <div className="font-semibold text-orange-600 text-base">{f.title}</div>
                                <div className="text-sm text-orange-400">{f.subtitle}</div>
                            </div>
                        </div>
                    );
                })}
{/* RIGHT FEATURES */}
                {rightFeatures.map((f, i) => {
                    const { tickX, tickY } = tickPos(i, rightFeatures.length, "right");
                    return (
                        <div key={i} className="absolute">
                            <div
                                style={{
                                    left: tickX,
                                    top: tickY,
                                    transform: "translate(-50%, -50%)",
                                    position: "absolute",
                                }}
                                className="w-3 h-3 rounded-full bg-orange-500 border border-white"
                            ></div>
                            <div
                                style={{
                                    left: tickX + 50,
                                    top: tickY,
                                    transform: "translate(0, -50%)",
                                    position: "absolute",
                                }}
                                className="text-left"
                            >
                                <div className="font-semibold text-orange-600 text-base">{f.title}</div>
                                <div className="text-sm text-orange-400">{f.subtitle}</div>
                            </div>
                        </div>
                    );
                })}

                {/* CENTER BOTTLE - Floating */}
                <div
                    className="absolute inset-0 flex items-center justify-center pointer-events-none"
                >
                    <motion.img
                        src={benefits.benifitImg}
                        alt="Nutri Bears Bottle"
                        className="drop-shadow-2xl"
                        style={{
                            width: "420px",
                            height: "auto",
                            objectFit: "contain",
                        }}
                        initial={{ y: 0, rotate: 0 }}
                        animate={{ y: [0, -12, 0], rotate: [0, 2, -2, 0] }}
                        transition={{ duration: 4, repeat: Infinity, ease: "easeInOut" }}
                    />
                </div>
            </div>
        </div>
    );
}
